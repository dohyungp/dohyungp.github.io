---
layout: default
title: FastAPI dp AWS Secrets Manager로 시크릿 우아하게 주입하기
---

AWS Secrets Manager에 환경변수를 저장해두고 원격에서 Cloud Config 주입하듯 주입시키고 싶어져서 좀 고민하다가,

```py
import json
from typing import Any

from botocore.exceptions import ClientError
import boto3
from pydantic import BaseSettings
from pydantic.env_settings import SettingsSourceCallable

from app.libs.metaclasses import Singleton


class SecretsManager(metaclass=Singleton):
    """시크릿 매니저"""

    _service_name = "secretsmanager"

    def __init__(self, region_name: str = "ap-northeast-2") -> None:
        session = boto3.Session()
        self.client = session.client(
            service_name=self._service_name,
            region_name=region_name,
        )

    def get_secret_value(self, secret_id: str):
        """시크릿 값 가져오기"""
        return self.client.get_secret_value(SecretId=secret_id)


class SecretManagerConfig:
    """시크릿 매니저"""

    secret_name: str = None

    @classmethod
    def _get_secrets_from_aws(cls, secret_name: str, default: Any | None = None) -> str | dict[str, Any]:
        secrets_manager = SecretsManager()
        try:
            secret_string = secrets_manager.get_secret_value(secret_id=secret_name)["SecretString"]
            return json.loads(secret_string)
        except (ClientError, json.decoder.JSONDecodeError):
            return default

    @classmethod
    def get_secrets(cls, settings: BaseSettings) -> dict[str, Any]:
        """시크릿 가져오기"""

        secrets = {}

        remote_secrets = cls._get_secrets_from_aws(cls.secret_name, {})

        for name, value in settings.__fields__.items():
            secrets[name] = remote_secrets.get(name) or value.default
        return secrets

    @classmethod
    def customise_sources(
        cls,
        init_settings: SettingsSourceCallable,
        env_settings: SettingsSourceCallable,
        file_secret_settings: SettingsSourceCallable,
    ):
        """커스텀 소스"""
        return (
            init_settings,
            cls.get_secrets,
            env_settings,
            file_secret_settings,
        )
```


이렇게 하면 좀 우아하게 가져올 수 있겠다 싶었다.

이는 실제로 아래와 같이 써먹을 수 있다.

```py
def _format_secret_name(secret_name):
    env = os.getenv("ENV", "local")
    return f"{secret_name}/{env}".lower()


class EnvType(str, Enum):
    """개발 환경"""

    LOCAL = "local"
    DEV = "dev"
    PROD = "prod"


class Settings(BaseSettings):
    """개발환경 세팅"""

    APP_NAME: str = "앱이름"
    APP_ORIGIN: str = "https://test.it"
    ENV: EnvType = EnvType.LOCAL
    SENTRY_DSN: str | None = None
    REDIS_URL: str
    CACHE_KEY_PREFIX: str = "cache-key"
    DB_READER_URI: str
    DB_WRITER_URI: str

    class Config(SecretManagerConfig):  # pylint: disable=missing-class-docstring
        case_sensitive = False
        secret_name = _format_secret_name("my-app")
```
