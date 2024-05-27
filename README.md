# Эксперимент с колоночным хранением в PostgreSQL

## Установка
Для установки собственно PostgreSQL и расширения TimescaleDB в Ubuntu 22 выполните следующие команды (инструкция взята отсюда: https://docs.timescale.com/self-hosted/latest/install/installation-linux/) :
```
sudo apt install gnupg postgresql-common apt-transport-https lsb-release wget

sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh

echo "deb https://packagecloud.io/timescale/timescaledb/ubuntu/ $(lsb_release -c -s) main" | sudo tee /etc/apt/sources.list.d/timescaledb.list

wget --quiet -O - https://packagecloud.io/timescale/timescaledb/gpgkey | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/timescaledb.gpg

sudo apt update

sudo apt install timescaledb-2-postgresql-16 postgresql-client
```
После этого запустите скрипт "тюнинга" параметров, в котором отвечайде yes на все вопросы:
```
sudo timescaledb-tune
```
Перезапуск для применения параметров
```
sudo systemctl restart postgresql
```

## Инициализация БД
Вход в консоль Постгреса:
```
sudo -u postgres psql
```
В консоли Постгреса нужно создать базу данных для тестов:
```
create database benchtest;
```
И выйти.

Наливаем базу тестовыми данными:
```
sudo -u postgres pgbench -i -s 200 benchtest
```
Сама по себе база будет крайне неинтересна, так как там слишком однородные данные. Запустим транзакционный тест и дадим ему поработать минут 10 (600 секунд если смотреть на параметры запуска). После этого данные станут разнообразнее и там будет что то видно:
```
sudo -u postgres pbgench -c 5 -T 600 -P 5 benchtest
```
Снова заходим в консоль постгреса:
```
sudo -u postgres psql
```
Переключаемся на нашу базу:
```
\c benchtest
```
Смотрим какие там таблицы есть и их размеры:
```
\d+
```

## Создаем колоночную таблицу
Чтобы работать с таблицами Timescale нужно включить расширение Постгреса:
```
create extension timescaledb;
```
Далее можно создавать колоночную таблицу как копию существующей:
```
create table accounts_columnar as SELECT * FROM  pgbench_accounts where 1=0;
```
Но пока мы просто создали таблицу, её теперь надо превратить в "гипертаблицу" в терминах TimescaleDB:
```
SELECT create_hypertable('accounts_columnar', 'abalance');
```
## Включаем компрессию и оцениваем эффективность
Чтобы воспользоваться всеми преимуществами, включим компрессию:
```
alter table accounts_columnar SET (timescaledb.compress, timescaledb.compress_segmentby='abalance');
```
Теперь зальем туда данные:
```
insert into accounts_columnar select * from pgbench_accounts;
```
К сожалению просто включить компрессию мало. Факт включения лишь обеспечивает в последствии автоматическое сжатие, однако прямо сейчас этого не произойдет, и чтобы все таки сжать данные, необходимо выполнить еще две команды. Первая покажет названия блоков, на которые разбита таблица. Их там будет 2:
```
SELECT show_chunks('accounts_columnar');
```
Далее к каждому блоку нужно применить сжатия прямо сейчас, указав название блока в команде:
```
SELECT compress_chunk('_timescaledb_internal._hyper_1_4_chunk');
```
Вот теперь можно смотреть эффективность хранения и запросов.
Включаем подсчет точного времени выполнения:
```
\timing on 
```
Для начала посмотрим сколько места занимают данные. Для обычных таблиц это видно в ответе на команду \d+
**Таблица pgbench_accounts весит 2600Мб**. 
Подсчет размера гипертаблицы делается через функцию:
```
SELECT pg_size_pretty(sum(after_compression_total_bytes)) AS total
  FROM chunk_compression_stats('acсounts_columnar');
```
Ответ: **27Мб. Эффективность почти в 100 раз!!!**
Это пожалуй черезчур оптимистично, тут дело в том что таблица слишком однородна, в реальности размер колоночной таблицы будет примерно в 5 раз меньше оригинального.

## Запросы. 
Простейший подсчет количества записей с ненулевым балансом:
```
select count(*) from pgbench_accounts where abalance>0;
```
На обычной таблице ответ выдается **~1000ms**
На колоночной:
```
select count(*) from accounts_columnar where abalance>0;
```
Время: **57ms - почти в 20 раз быстрее** 

Посмотреть топ 10 значений поля abalance по количеству среди ненулевых:
```
select abalance,count(*) from pgbench_accounts where abalance>0 group by abalance order by 2 desc limit 10;
```
На обычной таблице: **1160ms**

На колоночной:
```
select abalance,count(*) from ac2 where abalance>0 group by abalance order by 2 desc limit 10;
```

Время: **80ms - в 15 раз**
	
# Послесловие
Обращу ваше внимание, что эффективность запросов сильно зависит от того, какое поле выбрано для разбиения при указании сжатия на гипертаблицу. Выбор поля целиком и полностью зависит от бизнес задачи и структуры используемых запросов. Все сильно проще если у вас есть четко определенное поле, которое разбивает таблицу на независимые блоки. 
