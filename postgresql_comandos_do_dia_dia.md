# Comando usado no dia a dia na vida de um DBA PostgreSQL


[Serviço postfix fora do ar](#serviço-postfix-fora-do-ar-servico-postfix)

[Identificar tarefas que estão em execução a muito tempo](#identificar-tarefas-que-estão-em-execução-a-muito-tempo)




## Retorna as configurações do servidor:
````
SELECT name, setting FROM pg_settings
WHERE name = 'wal_level';


SELECT name, setting FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'maintenance_work_mem', 'effective_cache_size');

SELECT 
    name,
    setting,
    unit,
    CASE 
        WHEN name IN ('shared_buffers', 'work_mem', 'maintenance_work_mem', 'effective_cache_size') THEN 
            CASE 
                WHEN unit = '8kB' THEN (setting::bigint * 8192) / (1024 * 1024 * 1024) -- Convert 8kB to GB
                WHEN unit = 'kB' THEN (setting::bigint * 1024) / (1024 * 1024 * 1024) -- Convert kB to GB
                ELSE setting::bigint / (1024 * 1024 * 1024) -- Assume bytes for other units
            END
        ELSE NULL
    END AS setting_in_gb
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'maintenance_work_mem', 'effective_cache_size');

````








## Serviço postfix fora do ar {#servico-postfix}
Quando serviço do postfix apresenta erro "O serviço postfix (Postfix Mail Transport Agent) não está em execução"

### Procedimentos:
__1. Verificar se o Postfix está instalado__
Para verificar se o Postfix está instalado no sistema, você pode usar o comando dpkg (em sistemas baseados no Debian, como Ubuntu) ou rpm (em distribuições como CentOS, Fedora e RHEL).

+ Em sistemas Debian/Ubuntu:
````
dpkg -l | grep postfix
````
+ Em sistemas RHEL/CentOS:
````
rpm -qa | grep postfix
````

__2. Verificar o status do serviço Postfix__
Para verificar se o serviço Postfix está rodando, você pode usar o comando systemctl ou service.

Verificar o status com systemctl:

bash
Copiar código
````
sudo systemctl status postfix
````
Isso mostrará se o serviço está ativo e rodando. Se o Postfix estiver em execução, você verá uma saída indicando que ele está ativo (running).

Verificar com service:

bash
Copiar código
````
sudo service postfix status
````
Isso funciona em distribuições mais antigas que utilizam o gerenciamento de serviços tradicional.


Solução conhecida, para erro:
````
sudo systemctl status postfix
● postfix.service - Postfix Mail Transport Agent
   Loaded: loaded (/usr/lib/systemd/system/postfix.service; enabled; vendor preset: disabled)
   Active: inactive (dead)
[xxxxxxxx ~]$ sudo systemctl start postfix
Job for postfix.service failed because the control process exited with error code. See "systemctl status postfix.service" and "journalctl -xe" for details.
[jfilho.flowti@hsipacsbd01 ~]$ sudo journalctl -xe
-- Unit postfix.service has begun starting up.
Sep 27 15:10:44 hsipacsbd01 aliasesdb[10718]: /usr/sbin/postconf: fatal: parameter inet_interfaces: no local interface found for ::1
Sep 27 15:10:45 hsipacsbd01 aliasesdb[10718]: newaliases: fatal: parameter inet_interfaces: no local interface found for ::1
Sep 27 15:10:45 hsipacsbd01 postfix/sendmail[10722]: fatal: parameter inet_interfaces: no local interface found for ::1
Sep 27 15:10:45 hsipacsbd01 postfix[10728]: fatal: parameter inet_interfaces: no local interface found for ::1
Sep 27 15:10:46 hsipacsbd01 systemd[1]: postfix.service: control process exited, code=exited status=1
Sep 27 15:10:46 hsipacsbd01 systemd[1]: Failed to start Postfix Mail Transport Agent.
-- Subject: Unit postfix.service has failed
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
--
-- Unit postfix.service has failed.
--
-- The result is failed.
Sep 27 15:10:46 hsipacsbd01 systemd[1]: Unit postfix.service entered failed state.
Sep 27 15:10:46 hsipacsbd01 systemd[1]: postfix.service failed.
Sep 27 15:10:46 hsipacsbd01 polkitd[1070]: Unregistered Authentication Agent for unix-process:10712:1650631507 (system bus name :1.1785157, object path /org/freedesktop/PolicyKit1/Authenticatio
Sep 27 15:10:46 hsipacsbd01 sudo[10710]: pam_unix(sudo:session): session closed for user root
Sep 27 15:10:46 hsipacsbd01 systemd-logind[2002]: Removed session 37000.
````

__1. Modifique a configuração do Postfix__
Se você não precisar do suporte a IPv6 ou se não tiver uma interface configurada, você pode mudar a configuração do inet_interfaces no arquivo /etc/postfix/main.cf.

Abra o arquivo de configuração:

````
sudo nano /etc/postfix/main.cf
````
Localize a linha que contém inet_interfaces. Pode estar como:

````
inet_interfaces = all
````

Reinicie o serviço:
````
sudo systemctl start postfix
````


## Identificar tarefas que estão em execução a muito tempo

Retorna tarefas em execução com mais de 20 minutos:
````
SELECT 
datname AS database_name,
pid,
usename,
state,
now() - state_change AS idle_duration
FROM pg_stat_activity
WHERE state IN ('idle', 'idle in transaction', 'idle in transaction (aborted)', 'disabled')
AND now() - state_change > interval '20 minutes';
````

Comando para finalizar as tarefas em execução com mais de 20 minutos
````
SELECT 
pg_terminate_backend(pid), -- esta linha que mata os processos listado na query
datname AS database_name,
pid,
usename,
state,
now() - state_change AS idle_duration
FROM pg_stat_activity
WHERE state IN ('idle', 'idle in transaction', 'idle in transaction (aborted)', 'disabled');
AND now() - state_change > interval '120 minutes';
````


## Retorna locks
````
WITH waiting_locks AS (
    SELECT 
        a.pid AS waiting_pid,
        a.usename AS waiting_user,
        a.query AS waiting_query,
        a.state,
        a.application_name,
        a.backend_start,
        a.query_start,
        l.locktype,
        l.database,
        l.relation,
        l.page,
        l.tuple,
        l.virtualxid,
        l.transactionid,
        l.virtualtransaction
    FROM 
        pg_catalog.pg_stat_activity a
    JOIN 
        pg_catalog.pg_locks l ON a.pid = l.pid
    WHERE 
        NOT l.granted
),
blocking_locks AS (
    SELECT 
        a.pid AS blocking_pid,
        a.usename AS blocking_user,
        a.query AS blocking_query,
        l.locktype,
        l.database,
        l.relation,
        l.page,
        l.tuple,
        l.virtualxid,
        l.transactionid,
        l.virtualtransaction
    FROM 
        pg_catalog.pg_stat_activity a
    JOIN 
        pg_catalog.pg_locks l ON a.pid = l.pid
    WHERE 
        l.granted
)
SELECT 
    bl.blocking_pid,
    bl.blocking_user,
    bl.blocking_query,
    wl.waiting_pid,
    wl.waiting_user,
    wl.waiting_query,
    wl.state,
    wl.application_name,
    wl.backend_start,
    wl.query_start,
    wl.locktype
FROM 
    waiting_locks wl
JOIN 
    blocking_locks bl ON 
    wl.locktype = bl.locktype AND 
    wl.database IS NOT DISTINCT FROM bl.database AND 
    wl.relation IS NOT DISTINCT FROM bl.relation AND 
    wl.page IS NOT DISTINCT FROM bl.page AND 
    wl.tuple IS NOT DISTINCT FROM bl.tuple AND 
    wl.virtualxid IS NOT DISTINCT FROM bl.virtualxid AND 
    wl.transactionid IS NOT DISTINCT FROM bl.transactionid AND 
    wl.virtualtransaction IS NOT DISTINCT FROM bl.virtualtransaction;
````