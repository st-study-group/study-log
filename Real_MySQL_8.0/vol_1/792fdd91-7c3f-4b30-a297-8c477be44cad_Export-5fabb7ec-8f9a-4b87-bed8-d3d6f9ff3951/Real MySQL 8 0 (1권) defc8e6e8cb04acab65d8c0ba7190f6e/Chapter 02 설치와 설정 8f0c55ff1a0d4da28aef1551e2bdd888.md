# Chapter 02 설치와 설정

### 2.1.1 버전과 에디션 선택

MySQL 서버의 버전을 선택할 때는 다른 제약 사항이 없다면 가능한 최신 버전을 설치하는것이 좋음

기존 버전에서 새로운 메이저 버전으로 업그레이드 하는 경우라면 최소 패치 버전이 15~20번 이상 릴리스 된 버전을 선택하는 것이 안정적인 서비스에 도움됨

MySQL서버의 상용화 전략은 핵심 내용은 엔터프라이즈 에디션과 커뮤니티 에디션 모두 동일하며, 특정 부가 기능들만 상용 버전인 엔터프라이즈 에디션에 포함되는 방식임.

이러한 상용화 방식을 오픈 코어 모델 이라고함.

엔터프라이즈 에디션과 커뮤니티 에디션의 핵심 기능은 거의 차이가 없음.

다음과 같은 부가적인 기능과 서비스들은 엔터프라이즈 에디션에서만 지원됨

- Thread Pool
- Enterprise Audit
- Enterprise TDE(Master Key 관리)
- Enterprise Authentication
- Enterprise Firewall
- Enterprise Monitor
- Enterprise Backup
- MySQL 기술 지원

경험상 Percona에서 출시하는 Percona Server 백업 및 모니터링 도구 또는 Percona Server에서 지원하는 플로그인을 활용하면 MySQL 커뮤니티 에디션의 부족한 부분을 메꿀 수 있음.

### 2.2.2 시작과 종료

linux 사용시 시작, 상태, 중지

```bash
# 시작
systemctl start mysqld

# 상태
systemctl status mysqld

# 프로세스 확인
ps -ef | grep mysqld

# 중지
systemctl stop mysqld
```

mysql 로그인 후

```bash
# 서버 셧다운
mysql > SHUTDOWN;

# MySQL 서버가 종료될때 모든 커밋된 내용을 데이터 파일에 기록하고 종료하는 법
mysql > SET GLOBAL innodb_fast_shutdown=0;
linux > systemctl stop mysqld.service

## mysql 접속시
mysql > SET GLOBAL innodb_fast_shutdown=0;
mysql > SHUTDOWN;
```

모든 커밋된 데이터를 데이터 파일에 적용하고 종료하는 것을 클린셧다운 이라고 함.

클린 셧다운으로 종료되면 다시 MySQL 서버가 가동할 때 별도의 트랜잭션 복구 과정을 진행하지 않기 때문에 빠르기 시작할 수 있음.

### 2.2.3 서버 연결 테스트

```bash
linux > mysql -uroot -p --host=localhost --socket=/tmp/mysql.sock
linux > mysql -uroot -p --host=127.0.0.1 --port=3306
linux > mysql -uroot -p
```

### 2.3 MySQL 서버 업그레이드

MySQL 서버를 업그레이드하는 방법은 두 가지 방법을 생각 해 볼 수 있음.

1. MySQL 서버의 데이터 파일을 그대로 두고 업그레이드 하는 방법
2. mysqldump 도구 등을 이용해 MySQL 서버의 데이터를 SQL문장이나 텍스트 파일로 덤프한 후, 새로 업그레이드된 버전의 MySQL 서버에서 덤프된 데이터를 적재하는 방법

1번 방법을 ‘인플레이스 업그레이드’ 라고 함.

- 여러 가지 제약 사항이 있지만 업그레이드 시간을 크게 단축 할 수있음.

2번 방법을 ‘논리적 업그레이드’ 라고 함.

- 버전간 제약 사항이 거의 없지만 업그레이드 시간이 매우 많이 소요될 수 있음.

논리적 업그레이드 방법은 제외하고 인플레이스 업그레드의 제약사항을 살펴보고, MySQL 8.0으로 업그레이드 하는 방법에대해 살펴볼 예정

### 2.3.1 인플레이스 업그레이드 제약 사항

업그레이드는 마이너 버전간 업그레이드와 메이저 버전간 업그레이드가 있음.

동일 메이저 버전에서 마이너 버전간 업그레이드는 대부분 데이터 파일 변경없이 진행되며, 많은 경우 여러 버전을 건너뛰어서 업그레이드 하는것도 허용됨 (ex) 8.0.16 → 8.0.21)

메이저 버전간 업그레이드는 대부분 크고 작은 데이터 파일의 변경이 필요하기 때문에 반드시 직전 버전에서만 업그레이드가 허용됨 (ex) 5.5 → 5.6) 
두 단계를 건너뛰는 업그레이드는 지원하지 않음.

메이저 버전 업그레이드는 데이터 파일의 패치가 필요한데, Mysql 8.0서버의 프로그램은 직전 메이저 버전인 5.7 버전에서 사용하던 데이터 파일과 로그 포맷만 인식하도록 구현되어 있음.

만약 5.1 서버를 8.0 으로 업그레이드 해야한다면 5.1 → 5.5 → 5.6 → 5.7 → 8.0 순서로 업그레이드를 진행해야함. 상당히 번거로움
만약 두 단계 이상 업그레이드 해야한다면, mysqldump프로그램으로 MySQL서버에서 데이터를 백업받은 후 새로 구축된 MySQL 8.0 서버에서 데이터를 적재하는 ‘논리적 업그레이드’가 더 나은 방법일 수 있음.

메이저 버전 업그레이드가 특정 마이너 버전에서만 가능한 경우도 있음. 
5.7.8 → 8.0 은 불가함. 5.7.8이 GA(General Avaliablity)버전이 아니기 때문. GA 버전은 오라클에서 MySQL서버의 안정성이 확인된 버전이라는 것을 의미함. 
Mysql 서버를 선택할때 최소 GA버전은 지나서 15~20 번 이상의 마이너 버전의 선택하는것이 좋음.

따라서, 항상 메이저 버전 업그레이드를 할때 MySQL 서버 메뉴얼 정독 권장함.

### 2.3.2 MySQL 8.0 업그레이드 시 고려사항

MySQL 8.0에서 상당히 많은 기능들이 개선되거나 변경되었음.

기본적인 부분의 차이점과 8.0에서는 사용할수 없는 기능들이 몇가지 있음.
반드시 8.0 업그레이드하기 전에 아래 내용이 영향이 미치지 않는지 검토해야함.

- 사용자 인증방식: 기본인증 방식이 Caching SHA-2 Authentication인증 방식을 바뀜, 기존 호환되도록 변경하려면 옵션으로 파라미터 활성화 해야함.
- MySQL 8.0과 호환성 체크: 손상된 FRM파일(format(schema) file - 테이블 구조거 저장되어 있는 파일)이나 호환되지 않는 데이터 타입 또는 함수가 있는지 mysqlcheck 유틸리티로 확인해야함.
- 외래키 이름의 길이: 8.0 에서는 64자로 제한됨.
- 인덱스 힌트:  8.0 에서는 성능테스트를 해봐야함. (5.x 에서는 성능향상되었지만 8.0에서는 성능이 오히려 저하될 수 있음)
- Group By에 사용된 정렬 옵션: 5.x 에서는 group by절의 컬럼뒤에 asc,desc를 작성했다면 제거하거나 다른방식으로 변경해야함.
- 파티션을 위한 공용 데이블스페이스: 8.0 에서는 파티션의 각 테이블스페이스를 공용테이블스페이스에 저장할 수 없음. 파이션의 테이블스페이스가 공용테이블스페이스에 저장된것이 있는지 확인하고, 사용하고 있다면 개별 테이블스페이스를 사용하도록 변경해야함.

```bash
# mysqlcheck 유틸리티 실행 방법
linux > mysqlcheck -u root -p --all-databases --check-upgrade

```

### 2.3.3 MySQL 8.0 업그레이드

5.7 → 8.0 업그레이드는 MySQL내부적으로 진행하는 업그레이드 과정은 상당히 복잡함.

대부적으로 8.0 버전부터는 시스템 테이블의 정보와 데이터 딕셔너리 정보의 포맷이 완전이 바뀌었음.

5.7 → 8.0 업그레이드는 크게 두가지 단계로 나뉘어 처리됨

1. 데이터 딕셔너리 업그레이드: 5.7 버전까지는 데이터 딕셔너리 정보가 FRM 확장자를 가진 파일로 보관되었음. 8.0에서는 InnoDB테이블로 저장되도록 개선됨(트랜잭션 지원)
기존의 FRM파일의 내용을 InnoDB 시스템 테이블로 저장함
2. 서버 업그레이드: MySQL의 서버의 시스템 데이터베이스의 테이블 구조를 MySQL8.0버전에 맞게 변경함. (performance_schema, information_schema, mysql의 데이터베이스)

<aside>
💡 8.0.15 버전 이전 업그레이드
1. MySQL 셧다운
2. MySQL 5.7 프로그램 삭제 
3. MySQL 8.0 프로그램 설치
4. MySQL 8.0 서버 시작(MySQL서버가 데이터 딕셔너리 업그레이드 자동 실행)
5. mysql_upgrade 프로그램 실행(시스템 테이블 구조 MySQL 8.0에 맞게 변경)

8.0.16 버전 이후 업그레이드 (mysql_upgrade 유틸리티가 없어지고 서버가 작업을 함)

1. MySQL 셧다운
2. MySQL 5.7 프로그램 삭제 
3. MySQL 8.0 프로그램 설치
4. MySQL 8.0 서버 시작(MySQL서버가 데이터 딕셔너리 업그레이드 자동 실행 후, 시스템 테이블 구조를 MySQL 8.0에 맞게 변경)

</aside>

### 2.4 서버 설정

일반적으로 MySQL 서버는 단 하나의 설정 파일을 사용함

- linux : my.cnf
- window : my.ini

다음과 같은 순서로 my.cnf 파일을 탐색함.

```bash
shell > mysqld --verbose --help
...
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf
...

shell > mysql --help
...
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf
..
```

MySQL 서버용 설정 파일은 1) /etc/my.cnf  2) /etc/mysql/my.cnf 주로 1), 2) 을 사용함.
하나의 장비에 2개 이상의 MySQL서버를 실행하는 경우에는 1번과 2번은 충돌이 발생할 수 있으므로 공유된 디렉터리가 아닌 별도 디렉터리에 설정파일을 준비하고, MySQL 시작 스크립트의 내용을 변경하는 방법을 사용함.

그러나 하나의 서버에 2개이상의 MySQL서버(인스턴스)를 실행하는 형태로 서비스 하는곳은 흔하지 않음(그랬던 경험도 없음)

### 2.4.1 설정 파일의 구성

MySQL 설정파일은 여러개의 설정 그룹을 담을 수 있음.

대체로 실행 프로그램 이름을 그룹명으로 사용함. 예를들어 mysqldump 프로그램은 [mysqldum]설정 그룹을, mysqld 프로그램은 [mysqld] 인 영역을 참조함.

이 설정 파일이 MySQL 서버만을 위한 설정파일 이라면 [mysqld] 그룹만 명시해도 무방함.

```yaml
[mysqld_safe]
malloc-lib = /opt/lib/libtcmalloc_minimal.so

[mysqld]
socket = /usr/local/mysql/tmp/mysql.sock
port = 3306

[mysql]
default-character-set = utf8mb4
socket = /usr/local/mysql/tmp/mysql.sock
port = 3304

[mysqldumb]
default-character-set = utf8mb4
socket = /usr/local/mysql/tmp/mysql.sock
port = 3305
```

### 2.4.2 MySQL 시스템 변수의 특징

MySQL서버는 기동하면서 설정 파일의 내용을 읽어 메모리나 작동 방식을 초기화하고, 접속된 사용자를 제어하기위해 이런한 값을 별도로 저장해둠.

MySQL 서버에서 이렇게 저장된 값을 시스템 변수 (System Variables)라고 함.

시스템 변수 값이 어떻게 MySQL 서버와 클라이언트에 영향을 미치는지 판단하려면 각 변수가 글로벌 변수 인지 세션변수 인지 구분할 수 있어야함.

시스템 변수가 가지는 5가지 속성의 의미는 다음과 같음

- Cmd-Line : 서버의 명령형 인자로 설정 될 수 있는지 여부를 나타냄
- Option file : MySQL의 설정파일인 my.cnf 로 제어할 수 있는지 여부를 나타냄
- System var : 시스템 환경 변수 인지 아닌지 나타냄
- Var Scope : 시스템 변수의 적용 범위를 나타냄, 이 시스템 변수가 영향 미치는 곳이 MySQL 서버 전체를 대상으로 하는지, 아니면 서버와 클라이언트 간의 커넥션만 인지 구분함.  어떤 변수는 세션과 글로벌 범위 둘다 적용 되기도함.
- Dynamic : 시스템 변수가 동적인지 정적인지 구분하는 변수임.

![스크린샷 2023-06-26 오후 11.40.51.png](Chapter%2002%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%8E%E1%85%B5%E1%84%8B%E1%85%AA%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%8C%E1%85%A5%E1%86%BC%208f0c55ff1a0d4da28aef1551e2bdd888/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-06-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_11.40.51.png)

```yaml
mysql > SHOW GLOBAL VARIABLES;
```

```yaml
mysql > SHOW GLOBAL VARIABLES;
-- aurora serverless 

Variable_name, Value
'activate_all_roles_on_login', 'OFF'
'admin_address', ''
'admin_port', '33062'
'admin_ssl_ca', ''
'admin_ssl_capath', ''
'admin_ssl_cert', ''
'admin_ssl_cipher', ''
'admin_ssl_crl', ''
'admin_ssl_crlpath', ''
'admin_ssl_key', ''
'admin_tls_ciphersuites', ''
'admin_tls_version', 'TLSv1,TLSv1.1,TLSv1.2'
'aurora_backtrace_dedupe_string_filename', 'backtrace_dedupe_strings.txt'
'aurora_binlog_reserved_event_bytes', '1024'
'aurora_caspian_disabled_scaling_components', ''
'aurora_connection_prepare_warning_threshold', '50000'
'aurora_connection_queued_time_warning_threshold', '2'
'aurora_das_persistence_threads', '0'
'aurora_disable_lambda_natives', 'OFF'
'aurora_enable_emergency_volume_growth', 'OFF'
'aurora_enable_repl_bin_log_filtering', 'ON'
'aurora_enable_staggered_replica_restart', 'OFF'
'aurora_full_double_precision_in_json', 'OFF'
'aurora_fwd_master_idle_timeout', '60'
'aurora_fwd_master_max_connections_pct', '10'
'aurora_fwd_writer_idle_timeout', '60'
'aurora_fwd_writer_max_connections_pct', '10'
'aurora_ignore_default_storage_engine_errors', 'OFF'
'aurora_load_from_s3_role', ''
'aurora_max_connections_limit', '16000'
'aurora_oom_response', ''
'aurora_parallel_query', 'OFF'
'aurora_performance_schema_sql_info_max_size', '4096'
'aurora_pq_force', 'OFF'
'aurora_read_replica_read_committed', 'OFF'
'aurora_replica_read_consistency', ''
'aurora_s3_chunk_size', '10485760'
'aurora_s3_upload_file_size_threshold', '6442450944'
'aurora_s3_upload_iocache_chunk_size', '0'
'aurora_s3_upload_stream_chunk_size', '10485760'
'aurora_select_into_s3_encryption_default', 'OFF'
'aurora_select_into_s3_role', ''
'aurora_server_id', 'storelink-prod-ro'
'aurora_use_key_prefetch', 'ON'
'aurora_version', '3.02.2'
'auto_generate_certs', 'ON'
'auto_increment_increment', '1'
'auto_increment_offset', '1'
'autocommit', 'ON'
'automatic_sp_privileges', 'ON'
'avoid_temporal_upgrade', 'OFF'
'aws_default_comprehend_role', ''
'aws_default_lambda_role', ''
'aws_default_s3_role', ''
'aws_default_sagemaker_role', ''
'awsauthenticationplugin_max_backoff_delay', '2000'
'awsauthenticationplugin_max_retry_count', '3'
'awsauthenticationplugin_retry_delay', '250'
'back_log', '300'
'basedir', '/rdsdbbin/oscar-8.0.mysql_aurora.3.02.2.0.18301.0/'
'big_tables', 'OFF'
'bind_address', '*'
'binlog_cache_size', '32768'
'binlog_checksum', 'CRC32'
'binlog_direct_non_transactional_updates', 'OFF'
'binlog_encryption', 'OFF'
'binlog_error_action', 'ABORT_SERVER'
'binlog_expire_logs_seconds', '0'
'binlog_format', 'ROW'
'binlog_group_commit_sync_delay', '0'
'binlog_group_commit_sync_no_delay_count', '0'
'binlog_gtid_simple_recovery', 'ON'
'binlog_max_flush_queue_time', '0'
'binlog_order_commits', 'ON'
'binlog_rotate_encryption_master_key_at_startup', 'OFF'
'binlog_row_event_max_size', '8192'
'binlog_row_image', 'FULL'
'binlog_row_metadata', 'MINIMAL'
'binlog_row_value_options', ''
'binlog_rows_query_log_events', 'OFF'
'binlog_stmt_cache_size', '32768'
'binlog_transaction_compression', 'OFF'
'binlog_transaction_compression_level_zstd', '3'
'binlog_transaction_dependency_history_size', '25000'
'binlog_transaction_dependency_tracking', 'COMMIT_ORDER'
'block_encryption_mode', 'aes-128-ecb'
'bulk_insert_buffer_size', '8388608'
'caching_sha2_password_auto_generate_rsa_keys', 'ON'
'caching_sha2_password_private_key_path', 'private_key.pem'
'caching_sha2_password_public_key_path', 'public_key.pem'
'character_set_client', 'utf8mb4'
'character_set_connection', 'utf8mb4'
'character_set_database', 'utf8mb4'
'character_set_filesystem', 'binary'
'character_set_results', 'utf8mb4'
'character_set_server', 'utf8mb4'
'character_set_system', 'utf8'
'character_sets_dir', '/rdsdbbin/oscar-8.0.mysql_aurora.3.02.2.0.18301.0/share/charsets/'
'check_proxy_users', 'OFF'
'collation_connection', 'utf8mb4_0900_ai_ci'
'collation_database', 'utf8mb4_0900_ai_ci'
'collation_server', 'utf8mb4_0900_ai_ci'
'completion_type', 'NO_CHAIN'
'concurrent_insert', 'AUTO'
'connect_timeout', '10'
'core_file', 'ON'
'create_admin_listener_thread', 'OFF'
'cte_max_recursion_depth', '1000'
'datadir', '/rdsdbdata/db/'
'default_authentication_plugin', 'mysql_native_password'
'default_collation_for_utf8mb4', 'utf8mb4_0900_ai_ci'
'default_password_lifetime', '0'
'default_storage_engine', 'InnoDB'
'default_tmp_storage_engine', 'MyISAM'
'default_week_format', '0'
'delay_key_write', 'ON'
'delayed_insert_limit', '100'
'delayed_insert_timeout', '300'
'delayed_queue_size', '1000'
'disabled_storage_engines', ''
'disconnect_on_expired_password', 'ON'
'div_precision_increment', '4'
'end_markers_in_json', 'OFF'
'enforce_gtid_consistency', 'OFF'
'eq_range_index_dive_limit', '200'
'event_scheduler', 'DISABLED'
'expire_logs_days', '0'
'explicit_defaults_for_timestamp', 'ON'
'external_log_file', '/tmp/mysql-external.log'
'flush', 'OFF'
'flush_time', '0'
'foreign_key_checks', 'ON'
'ft_boolean_syntax', '+ -><()~*:\"\"&|'
'ft_max_word_len', '84'
'ft_min_word_len', '4'
'ft_query_expansion_limit', '20'
'ft_stopword_file', '(built-in)'
'general_log', 'OFF'
'general_log_file', '/rdsdbdata/log/general/mysql-general.log'
'generated_random_password_length', '20'
'group_concat_max_len', '1024'
'group_replication_consistency', 'EVENTUAL'
'gtid_executed', ''
'gtid_executed_compression_period', '0'
'gtid_mode', 'OFF_PERMISSIVE'
'gtid_owned', ''
'gtid_purged', ''
'have_compress', 'YES'
'have_dynamic_loading', 'YES'
'have_geometry', 'YES'
'have_openssl', 'YES'
'have_profiling', 'YES'
'have_query_cache', 'NO'
'have_rtree_keys', 'YES'
'have_ssl', 'YES'
'have_statement_timeout', 'YES'
'have_symlink', 'DISABLED'
'histogram_generation_max_mem_size', '20000000'
'host_cache_size', '428'
'hostname', 'ip-172-23-1-7'
'information_schema_stats_expiry', '86400'
'init_connect', ''
'init_file', ''
'init_replica', ''
'init_slave', ''
'innodb_adaptive_flushing', 'ON'
'innodb_adaptive_flushing_lwm', '10'
'innodb_adaptive_hash_index', 'OFF'
'innodb_adaptive_hash_index_parts', '8'
'innodb_adaptive_max_sleep_delay', '150000'
'innodb_api_bk_commit_interval', '5'
'innodb_api_disable_rowlock', 'OFF'
'innodb_api_enable_binlog', 'OFF'
'innodb_api_enable_mdl', 'OFF'
'innodb_api_trx_level', '0'
'innodb_aurora_enable_auto_akp', 'OFF'
'innodb_autoextend_increment', '64'
'innodb_autoinc_lock_mode', '2'
'innodb_btr_prefetch_perf_optimization', 'ON'
'innodb_buffer_pool_chunk_size', '134217728'
'innodb_buffer_pool_dump_at_shutdown', 'OFF'
'innodb_buffer_pool_dump_now', 'OFF'
'innodb_buffer_pool_dump_pct', '25'
'innodb_buffer_pool_filename', 'ib_buffer_pool'
'innodb_buffer_pool_in_core_file', 'ON'
'innodb_buffer_pool_instances', '8'
'innodb_buffer_pool_load_abort', 'OFF'
'innodb_buffer_pool_load_at_startup', 'OFF'
'innodb_buffer_pool_load_now', 'OFF'
'innodb_buffer_pool_size', '294912000'
'innodb_change_buffer_max_size', '25'
'innodb_change_buffering', 'none'
'innodb_checksum_algorithm', 'none'
'innodb_cleanup_temp_tablespaces_in_background', 'ON'
'innodb_cmp_per_index_enabled', 'OFF'
'innodb_commit_concurrency', '0'
'innodb_compression_failure_threshold_pct', '5'
'innodb_compression_level', '6'
'innodb_compression_pad_pct_max', '50'
'innodb_concurrency_tickets', '5000'
'innodb_data_file_path', 'ibdata1:12M:autoextend'
'innodb_data_home_dir', ''
'innodb_deadlock_detect', 'ON'
'innodb_dedicated_server', 'OFF'
'innodb_default_row_format', 'dynamic'
'innodb_directories', ''
'innodb_disable_shm_reads', 'OFF'
'innodb_disable_sort_file_cache', 'OFF'
'innodb_doublewrite', 'OFF'
'innodb_doublewrite_batch_size', '0'
'innodb_doublewrite_dir', ''
'innodb_doublewrite_files', '0'
'innodb_doublewrite_pages', '0'
'innodb_extend_and_initialize', 'OFF'
'innodb_fast_shutdown', '1'
'innodb_file_per_table', 'ON'
'innodb_fill_factor', '100'
'innodb_flush_log_at_timeout', '1'
'innodb_flush_log_at_trx_commit', '1'
'innodb_flush_method', 'O_DIRECT'
'innodb_flush_neighbors', '0'
'innodb_flush_sync', 'ON'
'innodb_flushing_avg_loops', '30'
'innodb_force_load_corrupted', 'OFF'
'innodb_force_recovery', '0'
'innodb_fsync_threshold', '0'
'innodb_ft_aux_table', ''
'innodb_ft_cache_size', '8000000'
'innodb_ft_enable_diag_print', 'OFF'
'innodb_ft_enable_stopword', 'ON'
'innodb_ft_max_token_size', '84'
'innodb_ft_min_token_size', '3'
'innodb_ft_num_word_optimize', '2000'
'innodb_ft_result_cache_limit', '2000000000'
'innodb_ft_server_stopword_table', ''
'innodb_ft_sort_pll_degree', '2'
'innodb_ft_total_cache_size', '640000000'
'innodb_ft_user_stopword_table', ''
'innodb_idle_flush_pct', '100'
'innodb_io_capacity', '200'
'innodb_io_capacity_max', '2000'
'innodb_lock_wait_timeout', '50'
'innodb_log_buffer_size', '16777216'
'innodb_log_checksums', 'ON'
'innodb_log_compressed_pages', 'ON'
'innodb_log_file_size', '50331648'
'innodb_log_files_in_group', '2'
'innodb_log_group_home_dir', './'
'innodb_log_spin_cpu_abs_lwm', '80'
'innodb_log_spin_cpu_pct_hwm', '50'
'innodb_log_wait_for_flush_spin_hwm', '400'
'innodb_log_write_ahead_size', '8192'
'innodb_log_writer_threads', 'ON'
'innodb_lru_scan_depth', '1024'
'innodb_max_dirty_pages_pct', '90.000000'
'innodb_max_dirty_pages_pct_lwm', '10.000000'
'innodb_max_purge_lag', '0'
'innodb_max_purge_lag_delay', '0'
'innodb_max_undo_log_size', '1073741824'
'innodb_monitor_disable', ''
'innodb_monitor_enable', ''
'innodb_monitor_reset', ''
'innodb_monitor_reset_all', ''
'innodb_old_blocks_pct', '37'
'innodb_old_blocks_time', '1000'
'innodb_online_alter_log_max_size', '134217728'
'innodb_open_files', '6000'
'innodb_optimize_fulltext_only', 'OFF'
'innodb_page_cleaners', '4'
'innodb_page_size', '16384'
'innodb_parallel_read_threads', '4'
'innodb_print_all_deadlocks', 'OFF'
'innodb_print_ddl_logs', 'OFF'
'innodb_purge_batch_size', '3600'
'innodb_purge_rseg_truncate_frequency', '128'
'innodb_purge_threads', '6'
'innodb_random_read_ahead', 'OFF'
'innodb_read_ahead_threshold', '56'
'innodb_read_io_threads', '1'
'innodb_read_only', 'ON'
'innodb_redo_log_archive_dirs', ''
'innodb_redo_log_encrypt', 'OFF'
'innodb_replication_delay', '0'
'innodb_rollback_on_timeout', 'OFF'
'innodb_rollback_segments', '128'
'innodb_shared_buffer_pool_uses_huge_pages', 'ON'
'innodb_sort_buffer_size', '1048576'
'innodb_spin_wait_delay', '6'
'innodb_spin_wait_pause_multiplier', '50'
'innodb_stats_auto_recalc', 'ON'
'innodb_stats_include_delete_marked', 'OFF'
'innodb_stats_method', 'nulls_equal'
'innodb_stats_on_metadata', 'OFF'
'innodb_stats_persistent', 'ON'
'innodb_stats_persistent_sample_pages', '20'
'innodb_stats_transient_sample_pages', '8'
'innodb_status_output', 'OFF'
'innodb_status_output_locks', 'OFF'
'innodb_strict_mode', 'ON'
'innodb_sync_array_size', '1'
'innodb_sync_spin_loops', '30'
'innodb_table_locks', 'ON'
'innodb_temp_data_file_path', 'ibtmp1:12M:autoextend'
'innodb_temp_tablespaces_dir', './'
'innodb_thread_concurrency', '0'
'innodb_thread_sleep_delay', '10000'
'innodb_tmpdir', ''
'innodb_undo_directory', './'
'innodb_undo_log_encrypt', 'OFF'
'innodb_undo_log_truncate', 'OFF'
'innodb_undo_tablespaces', '2'
'innodb_use_native_aio', 'OFF'
'innodb_validate_tablespace_paths', 'OFF'
'innodb_version', '8.0.23'
'innodb_write_io_threads', '4'
'interactive_timeout', '28800'
'internal_tmp_mem_storage_engine', 'TempTable'
'join_buffer_size', '262144'
'keep_files_on_create', 'OFF'
'key_buffer_size', '16777216'
'key_cache_age_threshold', '300'
'key_cache_block_size', '1024'
'key_cache_division_limit', '100'
'keyring_operations', 'ON'
'large_files_support', 'ON'
'large_page_size', '0'
'large_pages', 'OFF'
'lc_messages', 'en_US'
'lc_messages_dir', '/rdsdbbin/oscar-8.0.mysql_aurora.3.02.2.0.18301.0/share/'
'lc_time_names', 'en_US'
'license', 'GPL'
'local_infile', 'ON'
'lock_wait_timeout', '31536000'
'locked_in_memory', 'OFF'
'log_bin', 'OFF'
'log_bin_basename', ''
'log_bin_index', ''
'log_bin_trust_function_creators', 'OFF'
'log_bin_use_v1_row_events', 'OFF'
'log_error', '/rdsdbdata/log/error/mysql-error.log'
'log_error_services', 'log_filter_internal; log_sink_internal'
'log_error_suppression_list', ''
'log_error_verbosity', '3'
'log_output', 'FILE'
'log_queries_not_using_indexes', 'OFF'
'log_raw', 'OFF'
'log_replica_updates', 'OFF'
'log_slave_updates', 'OFF'
'log_slow_admin_statements', 'OFF'
'log_slow_extra', 'OFF'
'log_slow_replica_statements', 'OFF'
'log_slow_slave_statements', 'OFF'
'log_statements_unsafe_for_binlog', 'ON'
'log_throttle_queries_not_using_indexes', '0'
'log_timestamps', 'UTC'
'long_query_time', '10.000000'
'low_priority_updates', 'OFF'
'lower_case_file_system', 'OFF'
'lower_case_table_names', '0'
'mandatory_roles', ''
'master_info_repository', 'TABLE'
'master_verify_checksum', 'OFF'
'max_allowed_packet', '67108864'
'max_binlog_cache_size', '18446744073709547520'
'max_binlog_size', '134217728'
'max_binlog_stmt_cache_size', '18446744073709547520'
'max_connect_errors', '100'
'max_connections', '300'
'max_delayed_threads', '20'
'max_digest_length', '1024'
'max_error_count', '1024'
'max_execution_time', '0'
'max_heap_table_size', '16777216'
'max_insert_delayed_threads', '20'
'max_join_size', '18446744073709551615'
'max_length_for_sort_data', '4096'
'max_points_in_geometry', '65536'
'max_prepared_stmt_count', '16382'
'max_relay_log_size', '0'
'max_seeks_for_key', '18446744073709551615'
'max_sort_length', '1024'
'max_sp_recursion_depth', '0'
'max_user_connections', '0'
'max_write_lock_count', '18446744073709551615'
'min_examined_row_limit', '0'
'myisam_data_pointer_size', '6'
'myisam_max_sort_file_size', '9223372036853727232'
'myisam_mmap_size', '18446744073709551615'
'myisam_recover_options', 'OFF'
'myisam_repair_threads', '1'
'myisam_sort_buffer_size', '8388608'
'myisam_stats_method', 'nulls_unequal'
'myisam_use_mmap', 'OFF'
'mysql_native_password_proxy_users', 'OFF'
'net_buffer_length', '16384'
'net_read_timeout', '30'
'net_retry_count', '10'
'net_write_timeout', '60'
'new', 'OFF'
'ngram_token_size', '2'
'offline_mode', 'OFF'
'old', 'OFF'
'old_alter_table', 'OFF'
'open_files_limit', '1048576'
'optimizer_prune_level', '1'
'optimizer_search_depth', '62'
'optimizer_switch', 'index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on,index_condition_pushdown=on,mrr=on,mrr_cost_based=on,block_nested_loop=on,batched_key_access=off,materialization=on,semijoin=on,loosescan=on,firstmatch=on,duplicateweedout=on,subquery_materialization_cost_based=on,use_index_extensions=on,condition_fanout_filter=on,derived_merge=on,use_invisible_indexes=off,skip_scan=on,hash_join=on,subquery_to_derived=off,prefer_ordering_index=on,hypergraph_optimizer=off,derived_condition_pushdown=on'
'optimizer_trace', 'enabled=off,one_line=off'
'optimizer_trace_features', 'greedy_search=on,range_optimizer=on,dynamic_range=on,repeated_subselect=on'
'optimizer_trace_limit', '1'
'optimizer_trace_max_mem_size', '1048576'
'optimizer_trace_offset', '-1'
'parser_max_mem_size', '18446744073709551615'
'partial_revokes', 'OFF'
'password_history', '0'
'password_require_current', 'OFF'
'password_reuse_interval', '0'
'performance_schema', 'ON'
'performance_schema__auto__', 'ON'
'performance_schema_accounts_size', '-1'
'performance_schema_digests_size', '5000'
'performance_schema_error_size', '4880'
'performance_schema_events_stages_history_long_size', '1000'
'performance_schema_events_stages_history_size', '10'
'performance_schema_events_statements_history_long_size', '1000'
'performance_schema_events_statements_history_size', '10'
'performance_schema_events_transactions_history_long_size', '1000'
'performance_schema_events_transactions_history_size', '10'
'performance_schema_events_waits_history_long_size', '1000'
'performance_schema_events_waits_history_size', '10'
'performance_schema_hosts_size', '-1'
'performance_schema_max_cond_classes', '100'
'performance_schema_max_cond_instances', '-1'
'performance_schema_max_digest_length', '1024'
'performance_schema_max_digest_sample_age', '60'
'performance_schema_max_file_classes', '80'
'performance_schema_max_file_handles', '32768'
'performance_schema_max_file_instances', '-1'
'performance_schema_max_index_stat', '-1'
'performance_schema_max_memory_classes', '450'
'performance_schema_max_metadata_locks', '-1'
'performance_schema_max_mutex_classes', '300'
'performance_schema_max_mutex_instances', '-1'
'performance_schema_max_prepared_statements_instances', '-1'
'performance_schema_max_program_instances', '-1'
'performance_schema_max_rwlock_classes', '60'
'performance_schema_max_rwlock_instances', '-1'
'performance_schema_max_socket_classes', '10'
'performance_schema_max_socket_instances', '-1'
'performance_schema_max_sql_text_length', '1024'
'performance_schema_max_stage_classes', '175'
'performance_schema_max_statement_classes', '223'
'performance_schema_max_statement_stack', '10'
'performance_schema_max_table_handles', '-1'
'performance_schema_max_table_instances', '-1'
'performance_schema_max_table_lock_stat', '-1'
'performance_schema_max_thread_classes', '100'
'performance_schema_max_thread_instances', '-1'
'performance_schema_session_connect_attrs_size', '512'
'performance_schema_setup_actors_size', '-1'
'performance_schema_setup_objects_size', '-1'
'performance_schema_show_processlist', 'OFF'
'performance_schema_users_size', '-1'
'persist_only_admin_x509_subject', ''
'persisted_globals_load', 'ON'
'pid_file', '/rdsdbdata/log/mysql-3306.pid'
'plugin_dir', '/rdsdbbin/oscar-8.0.mysql_aurora.3.02.2.0.18301.0/lib/plugin/'
'port', '3306'
'preload_buffer_size', '32768'
'print_identified_with_as_hex', 'OFF'
'profiling', 'OFF'
'profiling_history_size', '15'
'protocol_compression_algorithms', 'zlib,zstd,uncompressed'
'protocol_version', '10'
'query_alloc_block_size', '8192'
'query_prealloc_size', '8192'
'range_alloc_block_size', '4096'
'range_optimizer_max_mem_size', '8388608'
'rbr_exec_mode', 'STRICT'
'read_buffer_size', '262144'
'read_only', 'OFF'
'read_rnd_buffer_size', '524288'
'regexp_stack_limit', '8000000'
'regexp_time_limit', '32'
'relay_log', '/rdsdbdata/log/relaylog/relaylog'
'relay_log_basename', '/rdsdbdata/log/relaylog/relaylog'
'relay_log_index', '/rdsdbdata/log/relaylog/relaylog.index'
'relay_log_info_file', 'relay-log.info'
'relay_log_info_repository', 'TABLE'
'relay_log_purge', 'ON'
'relay_log_recovery', 'OFF'
'relay_log_space_limit', '1000000000'
'replica_allow_batching', 'OFF'
'replica_checkpoint_group', '512'
'replica_checkpoint_period', '300'
'replica_compressed_protocol', 'OFF'
'replica_exec_mode', 'STRICT'
'replica_load_tmpdir', '/rdsdbdata/tmp/'
'replica_max_allowed_packet', '1073741824'
'replica_net_timeout', '60'
'replica_parallel_type', 'DATABASE'
'replica_parallel_workers', '0'
'replica_pending_jobs_size_max', '134217728'
'replica_preserve_commit_order', 'OFF'
'replica_skip_errors', 'OFF'
'replica_sql_verify_checksum', 'ON'
'replica_transaction_retries', '10'
'replica_type_conversions', ''
'replication_optimize_for_static_plugin_config', 'OFF'
'replication_sender_observe_commit_only', 'OFF'
'report_host', ''
'report_password', ''
'report_port', '3306'
'report_user', ''
'require_secure_transport', 'OFF'
'rpl_read_size', '5242880'
'rpl_stop_replica_timeout', '31536000'
'rpl_stop_slave_timeout', '31536000'
'schema_definition_cache', '256'
'secondary_engine_cost_threshold', '100000.000000'
'secure_file_priv', '/secure_file_priv_dir/'
'select_into_buffer_size', '131072'
'select_into_disk_sync', 'OFF'
'select_into_disk_sync_delay', '0'
'server_audit_cw_upload', 'OFF'
'server_audit_events', ''
'server_audit_excl_users', ''
'server_audit_incl_users', ''
'server_audit_logging', 'OFF'
'server_audit_mode', '0'
'server_audit_query_log_limit', '65536'
'server_id', '0'
'server_id_bits', '32'
'server_uuid', 'a41a38f1-f824-3397-886f-256d482d6295'
'session_track_gtids', 'OFF'
'session_track_schema', 'ON'
'session_track_state_change', 'OFF'
'session_track_system_variables', 'time_zone,autocommit,character_set_client,character_set_results,character_set_connection'
'session_track_transaction_info', 'OFF'
'sha256_password_auto_generate_rsa_keys', 'ON'
'sha256_password_private_key_path', 'private_key.pem'
'sha256_password_proxy_users', 'OFF'
'sha256_password_public_key_path', 'public_key.pem'
'show_create_table_verbosity', 'OFF'
'show_old_temporals', 'OFF'
'skip_external_locking', 'ON'
'skip_name_resolve', 'ON'
'skip_networking', 'OFF'
'skip_show_database', 'OFF'
'slave_allow_batching', 'OFF'
'slave_checkpoint_group', '512'
'slave_checkpoint_period', '300'
'slave_compressed_protocol', 'OFF'
'slave_exec_mode', 'STRICT'
'slave_load_tmpdir', '/rdsdbdata/tmp/'
'slave_max_allowed_packet', '1073741824'
'slave_net_timeout', '60'
'slave_parallel_type', 'DATABASE'
'slave_parallel_workers', '0'
'slave_pending_jobs_size_max', '134217728'
'slave_preserve_commit_order', 'OFF'
'slave_rows_search_algorithms', 'INDEX_SCAN,HASH_SCAN'
'slave_skip_errors', 'OFF'
'slave_sql_verify_checksum', 'ON'
'slave_transaction_retries', '10'
'slave_type_conversions', ''
'slow_launch_time', '2'
'slow_query_log', 'OFF'
'slow_query_log_file', '/rdsdbdata/log/slowquery/mysql-slowquery.log'
'socket', '/tmp/mysql.sock'
'sort_buffer_size', '262144'
'source_verify_checksum', 'OFF'
'sql_auto_is_null', 'OFF'
'sql_big_selects', 'ON'
'sql_buffer_result', 'OFF'
'sql_log_off', 'OFF'
'sql_mode', ''
'sql_notes', 'ON'
'sql_quote_show_create', 'ON'
'sql_replica_skip_counter', '0'
'sql_require_primary_key', 'OFF'
'sql_safe_updates', 'OFF'
'sql_select_limit', '18446744073709551615'
'sql_slave_skip_counter', '0'
'sql_warnings', 'OFF'
'ssl_ca', '/rdsdbdata/rds-metadata/ca-cert.pem'
'ssl_capath', ''
'ssl_cert', '/rdsdbdata/rds-metadata/server-cert.pem'
'ssl_cipher', 'ECDH-ECDSA-AES128-GCM-SHA256:ECDH-RSA-AES128-SHA256:ADH-CAMELLIA128-SHA:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA:AES128-GCM-SHA256:DHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDH-ECDSA-AES256-GCM-SHA384:AES256-GCM-SHA384:AES256-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:ECDH-ECDSA-AES256-SHA384:ADH-AES256-SHA:AES256-SHA: ECDH-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA:AES128-SHA256:CAMELLIA256-SHA:ECDH-RSA-AES256-SHA384:ADH-CAMELLIA256-SHA:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-CHACHA20-POLY1305:ADH-AES256-SHA256:ECDH-ECDSA-AES128-SHA:ADH-AES128-SHA:AES128-SHA:ECDHE-RSA-AES128-SHA:ECDH-RSA-AES128-GCM-SHA256:DH-DSS-AES128-GCM-SHA256:ECDH-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDH-ECDSA-AES128-SHA256:CAMELLIA128-SHA:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:DHE-RSA-AES128-SHA256:ADH-AES128-GCM-SHA256:ADH-AES256-GCM-SHA384:DHE-RSA-AES256-SHA256:ECDH-RSA-AES256-SHA:E'
'ssl_crl', ''
'ssl_crlpath', ''
'ssl_fips_mode', 'OFF'
'ssl_key', '/rdsdbdata/rds-metadata/server-key.pem'
'stored_program_cache', '256'
'stored_program_definition_cache', '256'
'super_read_only', 'OFF'
'sync_binlog', '1'
'sync_master_info', '10000'
'sync_relay_log', '10000'
'sync_relay_log_info', '10000'
'sync_source_info', '10000'
'system_time_zone', 'UTC'
'table_definition_cache', '400'
'table_encryption_privilege_check', 'OFF'
'table_open_cache', '160'
'table_open_cache_instances', '16'
'tablespace_definition_cache', '256'
'temptable_max_mmap', '1073741824'
'temptable_max_ram', '1073741824'
'temptable_use_mmap', 'ON'
'thread_cache_size', '7'
'thread_handling', 'thread-pools'
'thread_stack', '262144'
'time_zone', 'Asia/Seoul'
'tls_ciphersuites', ''
'tls_version', 'TLSv1,TLSv1.1,TLSv1.2'
'tmp_table_size', '16777216'
'tmpdir', '/rdsdbdata/tmp/'
'transaction_alloc_block_size', '8192'
'transaction_isolation', 'REPEATABLE-READ'
'transaction_prealloc_size', '4096'
'transaction_read_only', 'OFF'
'transaction_write_set_extraction', 'XXHASH64'
'unique_checks', 'ON'
'updatable_views_with_limit', 'YES'
'user_disable_external_log', 'OFF'
'version', '8.0.23'
'version_comment', 'Source distribution'
'version_compile_machine', 'aarch64'
'version_compile_os', 'Linux'
'version_compile_zlib', '1.2.11'
'wait_timeout', '28800'
'windowing_use_high_precision', 'ON'
```

### 2.4.3 글로벌 변수와 세션 변수

일반적으로 세션별로 적용되는 시스템 변수의 경우 글로벌 변수 뿐만아니라 세션 변수에도 존재함.

이러한 경우 MySQL 메뉴얼의 ‘Var Scope’에는 ‘Both’라고 표시됨

- **글로벌 범위의 시스템 변수**는 하나의 MySQL 서버 인스턴스에서 전체적으로 영향을 미치는 시스템 변수를 의미함
대표적으로 InnoDB 버퍼 풀 크기(innodb_buffer_pool_size), MyISAM의 캐시크기(key_buffer_size)등이 대표적인 글로벌 영역의 시스템 변수임
- **세션범위의 시스템 변수**는 MySQL 클라이언트가 MySQL 서버에 접속할때 기본으로 부여하는 옵션의 기본값을 제어하는데 사용됨
클라이언트에 필요의 따라 개별 커넥션 단위로 다른 값으로 변경할 수 있는 것이 세션 변수임
**기본값은 글로벌 시스템 변수이며, 각 클라이언트가 가지는 값이 세션 변수임**
대표적으로는 autocommit 변수가 있음, 세션마다 ON,OFF 설정 할 수 있음, 한번 연결된 커넥션의 세션변수는 서버에서 강제로 변경할 수 없음
- 세션범위의 변수중 MySql 설정파일 (my.cnf)에 명시해 초기화 할 수 있는 변수는 대부분 범위가 ‘Both’임, 커넥션이 생성될때 커넥션의 기본값으로 사용되는 값임.
순수하게 범위가 세션이라고 명시된 시스템변수는 Mysql서버의 설정파일에 초깃값을 명시할 수 없으며, 커넥션이 만들어지는 순간부터 해당 커넥션에서만 유효한 설정 변수를 의미함.

### 2.4.4 정적 변수와 동적 변수

MySQL 서버가 기동중인 상태에서 변경 가능한지에 따라 동적변수와 정적변수 로 구분됨

디스크에 저장되어 있는 파일(my,cnf)을 변경하는 경우와 MySQL 서버의 메모리에 있는 시스템 변수를 변경하는 경우로 구분할 수 있음.

디스크 변경의 경우 MySQL 서버가 재시작 되어야 적용됨

일반적으로 글로벌 시스템 변수는 MySQL 서버의 기동중에는 변경할 수 없는 것이 많지만 실시간으로 변경 할 수 있는 것도 있음.

![스크린샷 2023-06-27 오전 12.03.19.png](Chapter%2002%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%8E%E1%85%B5%E1%84%8B%E1%85%AA%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%8C%E1%85%A5%E1%86%BC%208f0c55ff1a0d4da28aef1551e2bdd888/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-06-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_12.03.19.png)

```yaml
# SET 명령어로 값을 변경 할 수 있음
SET GLOBAL max_connection=500;
```

다만 SET 명령어로 값을 변경하는경우 설정파일(my.cnf) 파일에 반영되지 않아 기동중인 인스턴스에만 유효함.

영구히 적용하려면 my.cnf파일에 변경을 적용해야함.

MySQL 8.0 부터 SET PERSIST 명령을 통해 시스템변수와 설정파일을 동시에 설정이 가능하게 됨

SHOW 이나 SET 명령어에서 GLOBAL 키워드를 사용하면 글로벌 시스템 변수의 목록과 내용을 읽고 변경할 수 있음.

GLOBAL 을 빼면 자동으로 세션 변수를 조회하고 변경할 수 있음.

### 2.4.5 SET PERSIST

SET PERSIST 명령으로 시스템 변수를 변경하면 MySQL서버는 변경된 값을 즉시 적용하고 별도의 설정파일 (mysqld-auto.cnf)에 변경 내용을 추가로 기록해둠

MySQL 서버가 다시 시작될때 기본 설정 파일 (my.cnf)뿐만아니라 자동 생성된 mysqld-auto.cnf 파일을 같이 참조해서 시스템 변수를 적용함.

현재 실행중인 MySQL 서버에는 적용하지 않고 다음 재시작을 위해 mysqld-auto.cnf 파일에만 변경내용을 적용한다면 SET PERSIST_ONLY 명령을 사용하면됨

[mysqld-auto.cnf 파일 예제]

![스크린샷 2023-06-27 오전 12.26.58.png](Chapter%2002%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%8E%E1%85%B5%E1%84%8B%E1%85%AA%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%8C%E1%85%A5%E1%86%BC%208f0c55ff1a0d4da28aef1551e2bdd888/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-06-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_12.26.58.png)

특정 PERSIST 내용만 삭제해야하는 경우

RESET PERSIST ${variable} 로 삭제할 수 있음

파일을 직접 수정해서 설정하는것은 오류가 생길 수 있으니 PERSIST 명령어로 설정하는것을 권장함. 

### 2.4.6 my.cnf 파일

**MySQL 서버를 제대로 사용하려면 시스템 변수에 대한 이해가 상당히 많이 필요함.**

시스템 변수가 실행중인 서버의 하드웨어 특성과 서비스의 특성에 따라 오히려 성능을 떨어지게 만들 수도 있음

뒤에 각 장에서 시스템 변수에 대해 다룰 예정