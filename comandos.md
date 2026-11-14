# Histórico de Comandos - Cenário E

## 1. Inicialização do Primário
initdb -D primary -U postgres -A scram-sha-256 --pwfile=pw.txt
pg_ctl -D primary -l primary.log -o "-p 5442" -w start

## 2. Criação do usuário de replicação
psql -h 127.0.0.1 -p 5442 -U postgres -c "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'repl';"

## 3. Configuração da Réplica via pg_basebackup
pg_basebackup -h 127.0.0.1 -p 5442 -U replicator -D replica -R -X stream -C -S replica1_slot -P
pg_ctl -D replica -l replica.log -o "-p 5443" -w start

## 4. Teste de Replicação Síncrona e Travamento
psql -h 127.0.0.1 -p 5442 -U postgres -c "ALTER SYSTEM SET synchronous_standby_names = '*';" -c "SELECT pg_reload_conf();"
pg_ctl -D replica -m fast stop
psql -h 127.0.0.1 -p 5442 -U postgres -c "CREATE TABLE teste_sincrono (id serial); INSERT INTO teste_sincrono DEFAULT VALUES;"
psql -h 127.0.0.1 -p 5442 -U postgres -c "ALTER SYSTEM RESET synchronous_standby_names;" -c "SELECT pg_reload_conf();"
pg_ctl -D replica -l replica.log -o "-p 5443" -w start