---
layout: default
title: Airflow Scheduler Deeper look
---

## Airflow Scheduler 동작 로직 뜯어보기

Airflow가 버전이 올라가며 코드가 추상화되어 지금 안보면 나중에 더 모를듯하여 정리하는 글.

### Command 

Airflow Scheduler를 Bootstrap하기 위해 아래와 같은 명령어로 시작하게 된다.

```bash
airflow scheduler
```

실제 `docker-compose.yml` 을 읽어봐도 스케쥴러를 올리는건, scheduler 명령어 뿐이다.

```yaml
  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type SchedulerJob --hostname "$${HOSTNAME}"']
      interval: 10s
      timeout: 10s
      retries: 5
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully
```

그럼 scheduler 명령은 어떻게 시작할까? [airflow/cli/commands/scheduler_command.py](https://github.com/apache/airflow/blob/9fd54b7397/airflow/cli/commands/scheduler_command.py) 를 읽으면 좀 알 수 있는데, daemon의 DaemonContext를 통해 daemonzie되어 scheduler job을 구동하게 된다.

```py
@cli_utils.action_cli
def scheduler(args):
    """Starts Airflow Scheduler"""


    # ... 상략 ...

            ctx = daemon.DaemonContext(
                pidfile=TimeoutPIDLockFile(pid, -1),
                files_preserve=[handle],
                stdout=stdout_handle,
                stderr=stderr_handle,
                umask=int(settings.DAEMON_UMASK, 8),
            )
            with ctx:
                _run_scheduler_job(args=args)

    # ... 하략 ...
```

이 때 데몬 컨텍스트와 함께 동작하는 `_run_scheduler_job`은 `_create_scheduler_job`을 호출해 SchedulerJob 객체를 만든다.

```py
def _create_scheduler_job(args):
    job = SchedulerJob(
        subdir=process_subdir(args.subdir),
        num_runs=args.num_runs,
        do_pickle=args.do_pickle,
    )
    return job

def _run_scheduler_job(args):
    skip_serve_logs = args.skip_serve_logs
    job = _create_scheduler_job(args)
    logs_sub_proc = _serve_logs(skip_serve_logs)
    enable_health_check = conf.getboolean('scheduler', 'ENABLE_HEALTH_CHECK')
    health_sub_proc = _serve_health_check(enable_health_check)
    try:
        job.run()
    finally:
        if logs_sub_proc:
            logs_sub_proc.terminate()
        if health_sub_proc:
            health_sub_proc.terminate()
```

생성된 SchedulerJob은 run 메서드를 호출하게 되는데, run은 Abstract Class인 BaseJob의 메서드로 실제로는 `_execute` 메서드를 호출하도록 되어 있다. 이 _execute가 사실 상 airflow가 구동되는 로직을 다 담고 있다고 봐도 되는데, 

* Dag 파일을 파싱하는 `DagFileProcessorAgent` 구동
* `ExecutorLoader`를 통해 load되는 `executor` 구동
* `polling`하면서 스케쥴 구동


같은 로직들이 `_execute`로부터 시작한다.


```py
  def _execute(self) -> None:
        from airflow.dag_processing.manager import DagFileProcessorAgent

        self.log.info("Starting the scheduler")

        # DAGs can be pickled for easier remote execution by some executors
        pickle_dags = self.do_pickle and self.executor_class not in UNPICKLEABLE_EXECUTORS

        self.log.info("Processing each file at most %s times", self.num_times_parse_dags)

        # When using sqlite, we do not use async_mode
        # so the scheduler job and DAG parser don't access the DB at the same time.
        async_mode = not self.using_sqlite

        processor_timeout_seconds: int = conf.getint('core', 'dag_file_processor_timeout')
        processor_timeout = timedelta(seconds=processor_timeout_seconds)
        if not self._standalone_dag_processor:
            self.processor_agent = DagFileProcessorAgent(
                dag_directory=Path(self.subdir),
                max_runs=self.num_times_parse_dags,
                processor_timeout=processor_timeout,
                dag_ids=[],
                pickle_dags=pickle_dags,
                async_mode=async_mode,
            )

        try:
            self.executor.job_id = self.id
            if self.processor_agent:
                self.log.debug("Using PipeCallbackSink as callback sink.")
                self.executor.callback_sink = PipeCallbackSink(
                    get_sink_pipe=self.processor_agent.get_callbacks_pipe
                )
            else:
                from airflow.callbacks.database_callback_sink import DatabaseCallbackSink

                self.log.debug("Using DatabaseCallbackSink as callback sink.")
                self.executor.callback_sink = DatabaseCallbackSink()

            self.executor.start()

            self.register_signals()

            if self.processor_agent:
                self.processor_agent.start()

            execute_start_time = timezone.utcnow()

            self._run_scheduler_loop()

            if self.processor_agent:
                # Stop any processors
                self.processor_agent.terminate()

                # Verify that all files were processed, and if so, deactivate DAGs that
                # haven't been touched by the scheduler as they likely have been
                # deleted.
                if self.processor_agent.all_files_processed:
                    self.log.info(
                        "Deactivating DAGs that haven't been touched since %s", execute_start_time.isoformat()
                    )
                    DAG.deactivate_stale_dags(execute_start_time)

            settings.Session.remove()  # type: ignore
        except Exception:
            self.log.exception("Exception when executing SchedulerJob._run_scheduler_loop")
            raise
        finally:
            try:
                self.executor.end()
            except Exception:
                self.log.exception("Exception when executing Executor.end")
            if self.processor_agent:
                try:
                    self.processor_agent.end()
                except Exception:
                    self.log.exception("Exception when executing DagFileProcessorAgent.end")
            self.log.info("Exited execute loop")
```
