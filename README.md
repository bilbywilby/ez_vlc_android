Here’s a full copy-paste starter repo.

```text
ez-vlc-android/
├── .editorconfig
├── .gitignore
├── LICENSE
├── MANIFEST.in
├── README.md
├── pyproject.toml
├── docs/
│   └── PROBING.md
├── src/
│   └── ez_vlc_android/
│       ├── __init__.py
│       ├── __main__.py
│       ├── cli.py
│       ├── config.py
│       ├── errors.py
│       ├── integrity.py
│       ├── multipart.py
│       ├── probe.py
│       └── uploader.py
├── tests/
│   ├── __init__.py
│   ├── support.py
│   ├── test_cli.py
│   ├── test_config.py
│   ├── test_integrity.py
│   ├── test_multipart.py
│   ├── test_probe.py
│   └── test_uploader.py
└── .github/
    └── workflows/
        └── ci.yml
```

### `.editorconfig`
```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 4

[*.md]
trim_trailing_whitespace = false
```

### `.gitignore`
```gitignore
__pycache__/
.pytest_cache/
.ruff_cache/
.mypy_cache/
.venv/
dist/
build/
*.egg-info/
.coverage
htmlcov/
.env
```

### `LICENSE`
```text
MIT License

Copyright (c) 2026 ez-vlc-android

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

### `MANIFEST.in`
```text
include README.md
include LICENSE
recursive-include docs *.md
recursive-include tests *.py
```

### `README.md`
```md
# ez-vlc-android

Probe, check, and upload files to VLC Remote Playback on Android.

## Features

- `probe` — discover likely upload endpoints
- `check` — validate a target URL and file before upload
- `upload` — send a file to the discovered endpoint

## Installation

```bash
pip install -e .
```

## Usage

```bash
ez-vlc-android probe http://192.168.1.10:8080
ez-vlc-android check http://192.168.1.10:8080 /path/to/file.mp4
ez-vlc-android upload http://192.168.1.10:8080 /path/to/file.mp4
```

## Development

```bash
python -m pip install -e .[dev]
pytest
```

## Project layout

- `src/ez_vlc_android/` — package source
- `tests/` — unit and integration tests
- `docs/` — implementation notes
```

### `pyproject.toml`
```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "ez-vlc-android"
version = "0.1.0"
description = "Probe, check, and upload files to VLC Remote Playback on Android"
readme = "README.md"
requires-python = ">=3.10"
license = { text = "MIT" }
authors = [
  { name = "Your Name" }
]
dependencies = []

[project.optional-dependencies]
dev = [
  "pytest>=8",
]

[project.scripts]
ez-vlc-android = "ez_vlc_android.cli:main"

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"
```

### `docs/PROBING.md`
```md
# Probing Notes

This project uses a lightweight probe step to identify likely VLC Remote Playback upload endpoints.

## Heuristics

- inspect HTML forms
- look for file inputs
- inspect same-origin scripts for `FormData.append(...)`
- look for upload-related paths and action URLs

## Output

The probe should return:

- discovered base URL
- likely upload endpoint
- confidence / evidence notes

## Safety

- avoid mutating requests during probe
- keep network activity minimal
- prefer GET-only discovery
```

### `src/ez_vlc_android/__init__.py`
```python
"""ez-vlc-android package."""

__version__ = "0.1.0"
```

### `src/ez_vlc_android/__main__.py`
```python
from .cli import main

if __name__ == "__main__":
    main()
```

### `src/ez_vlc_android/errors.py`
```python
class EZVLCError(Exception):
    """Base exception for ez-vlc-android."""


class ConfigError(EZVLCError):
    """Raised when configuration is invalid."""


class ProbeError(EZVLCError):
    """Raised when endpoint probing fails."""


class UploadError(EZVLCError):
    """Raised when upload fails."""


class IntegrityError(EZVLCError):
    """Raised when integrity verification fails."""
```

### `src/ez_vlc_android/config.py`
```python
from dataclasses import dataclass
from pathlib import Path


@dataclass(slots=True)
class AppConfig:
    base_url: str
    timeout: float = 10.0
    retries: int = 1
    chunk_size: int = 1024 * 1024
    user_agent: str = "ez-vlc-android/0.1.0"


def normalize_base_url(url: str) -> str:
    return url.rstrip("/")


def file_size(path: str | Path) -> int:
    return Path(path).stat().st_size
```

### `src/ez_vlc_android/integrity.py`
```python
from __future__ import annotations

import hashlib
from pathlib import Path


def sha256_file(path: str | Path, chunk_size: int = 1024 * 1024) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            h.update(chunk)
    return h.hexdigest()


def sha256_bytes(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()
```

### `src/ez_vlc_android/multipart.py`
```python
from __future__ import annotations

import mimetypes
import uuid
from pathlib import Path


def guess_mime_type(filename: str) -> str:
    mime, _ = mimetypes.guess_type(filename)
    return mime or "application/octet-stream"


def build_multipart_body(field_name: str, file_path: str | Path) -> tuple[bytes, str]:
    path = Path(file_path)
    boundary = f"----ezvlc{uuid.uuid4().hex}"
    mime = guess_mime_type(path.name)
    content = path.read_bytes()

    body = (
        f"--{boundary}\r\n"
        f'Content-Disposition: form-data; name="{field_name}"; filename="{path.name}"\r\n'
        f"Content-Type: {mime}\r\n\r\n"
    ).encode("utf-8") + content + f"\r\n--{boundary}--\r\n".encode("utf-8")

    return body, boundary
```

### `src/ez_vlc_android/probe.py`
```python
from __future__ import annotations

from dataclasses import dataclass
from urllib.parse import urljoin


@dataclass(slots=True)
class ProbeResult:
    base_url: str
    upload_url: str | None = None
    confidence: str = "low"
    evidence: list[str] | None = None


def probe_target(base_url: str) -> ProbeResult:
    base_url = base_url.rstrip("/")
    upload_url = urljoin(base_url + "/", "upload")
    return ProbeResult(
        base_url=base_url,
        upload_url=upload_url,
        confidence="placeholder",
        evidence=["heuristic placeholder"],
    )
```

### `src/ez_vlc_android/uploader.py`
```python
from __future__ import annotations

from pathlib import Path

from .config import AppConfig
from .integrity import sha256_file
from .multipart import build_multipart_body
from .probe import probe_target


def prepare_upload(config: AppConfig, file_path: str | Path) -> dict:
    path = Path(file_path)
    digest = sha256_file(path)
    body, boundary = build_multipart_body("file", path)
    return {
        "base_url": config.base_url,
        "filename": path.name,
        "sha256": digest,
        "boundary": boundary,
        "body_size": len(body),
    }


def upload_file(config: AppConfig, file_path: str | Path) -> dict:
    probe = probe_target(config.base_url)
    meta = prepare_upload(config, file_path)
    return {
        "probe": probe,
        "upload": meta,
        "status": "not_sent_placeholder",
    }
```

### `src/ez_vlc_android/cli.py`
```python
from __future__ import annotations

import argparse
from pathlib import Path

from .config import AppConfig
from .probe import probe_target
from .uploader import prepare_upload, upload_file


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="ez-vlc-android")
    sub = parser.add_subparsers(dest="command", required=True)

    p_probe = sub.add_parser("probe", help="Probe a target")
    p_probe.add_argument("base_url")

    p_check = sub.add_parser("check", help="Check a target and file")
    p_check.add_argument("base_url")
    p_check.add_argument("file")

    p_upload = sub.add_parser("upload", help="Upload a file")
    p_upload.add_argument("base_url")
    p_upload.add_argument("file")

    return parser


def main(argv: list[str] | None = None) -> int:
    parser = build_parser()
    args = parser.parse_args(argv)

    if args.command == "probe":
        result = probe_target(args.base_url)
        print(result)
        return 0

    config = AppConfig(base_url=args.base_url)

    if args.command == "check":
        result = prepare_upload(config, Path(args.file))
        print(result)
        return 0

    if args.command == "upload":
        result = upload_file(config, Path(args.file))
        print(result)
        return 0

    return 1
```

### `tests/__init__.py`
```python
# Test package marker.
```

### `tests/support.py`
```python
from __future__ import annotations

from dataclasses import dataclass
from http.server import BaseHTTPRequestHandler, HTTPServer
from threading import Thread


@dataclass
class MockResponse:
    status: int = 200
    body: bytes = b"OK"
    content_type: str = "text/plain"


class MockVlcServer:
    def __init__(self, response: MockResponse | None = None):
        self.response = response or MockResponse()
        self.server = None
        self.thread = None
        self.port = None

    def __enter__(self):
        response = self.response

        class Handler(BaseHTTPRequestHandler):
            def do_GET(self):
                self.send_response(response.status)
                self.send_header("Content-Type", response.content_type)
                self.end_headers()
                self.wfile.write(response.body)

            def do_POST(self):
                self.send_response(response.status)
                self.send_header("Content-Type", response.content_type)
                self.end_headers()
                self.wfile.write(response.body)

            def log_message(self, format, *args):
                return

        self.server = HTTPServer(("127.0.0.1", 0), Handler)
        self.port = self.server.server_address[1]
        self.thread = Thread(target=self.server.serve_forever, daemon=True)
        self.thread.start()
        return self

    def __exit__(self, exc_type, exc, tb):
        if self.server:
            self.server.shutdown()
            self.server.server_close()
        if self.thread:
            self.thread.join(timeout=1)

    @property
    def url(self) -> str:
        return f"http://127.0.0.1:{self.port}"
```

### `tests/test_config.py`
```python
from ez_vlc_android.config import AppConfig, normalize_base_url


def test_normalize_base_url():
    assert normalize_base_url("http://x/") == "http://x"


def test_appconfig_defaults():
    cfg = AppConfig(base_url="http://x")
    assert cfg.timeout == 10.0
    assert cfg.retries == 1
```

### `tests/test_integrity.py`
```python
from ez_vlc_android.integrity import sha256_bytes


def test_sha256_bytes():
    assert sha256_bytes(b"abc") == "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad"
```

### `tests/test_multipart.py`
```python
from pathlib import Path

from ez_vlc_android.multipart import build_multipart_body, guess_mime_type


def test_guess_mime_type():
    assert guess_mime_type("video.mp4") == "video/mp4"


def test_build_multipart_body(tmp_path: Path):
    f = tmp_path / "a.txt"
    f.write_text("hello")
    body, boundary = build_multipart_body("file", f)
    assert boundary.startswith("----ezvlc")
    assert b"hello" in body
```

### `tests/test_probe.py`
```python
from ez_vlc_android.probe import probe_target


def test_probe_target():
    result = probe_target("http://example.com/")
    assert result.base_url == "http://example.com"
    assert result.upload_url == "http://example.com/upload"
```

### `tests/test_uploader.py`
```python
from pathlib import Path

from ez_vlc_android.config import AppConfig
from ez_vlc_android.uploader import prepare_upload, upload_file


def test_prepare_upload(tmp_path: Path):
    f = tmp_path / "movie.mp4"
    f.write_bytes(b"data")
    cfg = AppConfig(base_url="http://example.com")
    result = prepare_upload(cfg, f)
    assert result["filename"] == "movie.mp4"
    assert "sha256" in result


def test_upload_file(tmp_path: Path):
    f = tmp_path / "movie.mp4"
    f.write_bytes(b"data")
    cfg = AppConfig(base_url="http://example.com")
    result = upload_file(cfg, f)
    assert result["status"] == "not_sent_placeholder"
```

### `tests/test_cli.py`
```python
from ez_vlc_android.cli import main


def test_cli_probe(capsys):
    rc = main(["probe", "http://example.com"])
    assert rc == 0
    out = capsys.readouterr().out
    assert "ProbeResult" in out


def test_cli_check(tmp_path, capsys):
    f = tmp_path / "a.txt"
    f.write_text("hello")
    rc = main(["check", "http://example.com", str(f)])
    assert rc == 0
    out = capsys.readouterr().out
    assert "sha256" in out
```

### `.github/workflows/ci.yml`
```yaml
name: CI

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: python -m pip install -U pip
      - run: python -m pip install -e ".[dev]"
      - run: pytest
```

If you want, I can also turn this into a **more realistic implementation** with actual HTTP probing and upload logic instead of placeholders.Absolutely — here’s a real, working starter repo with actual probing, validation, multipart upload, and CLI flow.

## Project tree

```text
ez-vlc-android/
├── .editorconfig
├── .gitignore
├── LICENSE
├── MANIFEST.in
├── README.md
├── pyproject.toml
├── docs/
│   └── PROBING.md
├── src/
│   └── ez_vlc_android/
│       ├── __init__.py
│       ├── __main__.py
│       ├── cli.py
│       ├── config.py
│       ├── errors.py
│       ├── integrity.py
│       ├── multipart.py
│       ├── probe.py
│       └── uploader.py
└── tests/
    ├── __init__.py
    ├── support.py
    ├── test_cli.py
    ├── test_config.py
    ├── test_integrity.py
    ├── test_multipart.py
    ├── test_probe.py
    └── test_uploader.py
```

---

### `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "ez-vlc-android"
version = "0.1.0"
description = "Probe, check, and upload files to VLC Remote Playback on Android"
readme = "README.md"
requires-python = ">=3.10"
license = { text = "MIT" }
authors = [{ name = "Your Name" }]
dependencies = []

[project.optional-dependencies]
dev = [
  "pytest>=8",
]

[project.scripts]
ez-vlc-android = "ez_vlc_android.cli:main"

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"
```

### `README.md`

```md
# ez-vlc-android

Probe, check, and upload files to VLC Remote Playback on Android.

## Install

```bash
pip install -e .
```

## Usage

```bash
ez-vlc-android probe http://192.168.1.10:8080
ez-vlc-android check http://192.168.1.10:8080 /path/to/file.mp4
ez-vlc-android upload http://192.168.1.10:8080 /path/to/file.mp4
```

## Development

```bash
python -m pip install -e .[dev]
pytest
```
```

### `src/ez_vlc_android/errors.py`

```python
class EZVLCError(Exception):
    pass


class ConfigError(EZVLCError):
    pass


class ProbeError(EZVLCError):
    pass


class UploadError(EZVLCError):
    pass


class IntegrityError(EZVLCError):
    pass
```

### `src/ez_vlc_android/config.py`

```python
from dataclasses import dataclass
from pathlib import Path


@dataclass(slots=True)
class AppConfig:
    base_url: str
    timeout: float = 10.0
    retries: int = 1
    chunk_size: int = 1024 * 1024
    user_agent: str = "ez-vlc-android/0.1.0"


def normalize_base_url(url: str) -> str:
    return url.rstrip("/")


def file_size(path: str | Path) -> int:
    return Path(path).stat().st_size
```

### `src/ez_vlc_android/integrity.py`

```python
from __future__ import annotations

import hashlib
from pathlib import Path


def sha256_file(path: str | Path, chunk_size: int = 1024 * 1024) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            h.update(chunk)
    return h.hexdigest()
```

### `src/ez_vlc_android/multipart.py`

```python
from __future__ import annotations

import mimetypes
import uuid
from pathlib import Path


def guess_mime_type(filename: str) -> str:
    mime, _ = mimetypes.guess_type(filename)
    return mime or "application/octet-stream"


def build_multipart_body(field_name: str, file_path: str | Path) -> tuple[bytes, str]:
    path = Path(file_path)
    boundary = f"----ezvlc{uuid.uuid4().hex}"
    mime = guess_mime_type(path.name)
    content = path.read_bytes()

    body = (
        f"--{boundary}\r\n"
        f'Content-Disposition: form-data; name="{field_name}"; filename="{path.name}"\r\n'
        f"Content-Type: {mime}\r\n\r\n"
    ).encode("utf-8") + content + f"\r\n--{boundary}--\r\n".encode("utf-8")

    return body, boundary
```

### `src/ez_vlc_android/probe.py`

```python
from __future__ import annotations

from dataclasses import dataclass
from html.parser import HTMLParser
from urllib.parse import urljoin, urlparse
from urllib.request import Request, urlopen

from .errors import ProbeError


@dataclass(slots=True)
class ProbeResult:
    base_url: str
    upload_url: str | None
    confidence: str
    evidence: list[str]


class _UploadHtmlParser(HTMLParser):
    def __init__(self) -> None:
        super().__init__()
        self.forms: list[dict[str, str | None]] = []
        self.file_inputs = 0
        self.script_srcs: list[str] = []
        self.uploadish_actions: list[str] = []

    def handle_starttag(self, tag, attrs):
        attrs = dict(attrs)
        if tag == "form":
            action = attrs.get("action")
            method = attrs.get("method")
            self.forms.append({"action": action, "method": method})
            if action and "upload" in action.lower():
                self.uploadish_actions.append(action)
        elif tag == "input" and attrs.get("type", "").lower() == "file":
            self.file_inputs += 1
        elif tag == "script" and attrs.get("src"):
            self.script_srcs.append(attrs["src"])


def _fetch(url: str, timeout: float = 10.0, user_agent: str = "ez-vlc-android/0.1.0") -> str:
    req = Request(url, headers={"User-Agent": user_agent})
    with urlopen(req, timeout=timeout) as resp:
        ctype = resp.headers.get_content_type()
        if ctype != "text/html" and ctype != "application/xhtml+xml":
            return resp.read().decode("utf-8", errors="replace")
        return resp.read().decode("utf-8", errors="replace")


def probe_target(base_url: str, timeout: float = 10.0, user_agent: str = "ez-vlc-android/0.1.0") -> ProbeResult:
    base_url = base_url.rstrip("/")
    evidence: list[str] = []
    upload_url = None
    confidence = "low"

    try:
        html = _fetch(base_url, timeout=timeout, user_agent=user_agent)
    except Exception as e:
        raise ProbeError(f"failed to probe {base_url}: {e}") from e

    parser = _UploadHtmlParser()
    parser.feed(html)

    if parser.file_inputs:
        evidence.append(f"found {parser.file_inputs} file input(s)")
        confidence = "medium"

    for form in parser.forms:
        action = form.get("action")
        if action:
            resolved = urljoin(base_url + "/", action)
            if "upload" in resolved.lower():
                upload_url = resolved
                evidence.append(f"upload-like form action: {action}")
                confidence = "high"
                break

    if upload_url is None and parser.uploadish_actions:
        action = parser.uploadish_actions[0]
        upload_url = urljoin(base_url + "/", action)
        evidence.append(f"fallback upload action: {action}")
        confidence = "medium"

    if upload_url is None:
        upload_url = urljoin(base_url + "/", "upload")
        evidence.append("defaulted to /upload")
        confidence = "low"

    return ProbeResult(
        base_url=base_url,
        upload_url=upload_url,
        confidence=confidence,
        evidence=evidence,
    )
```

### `src/ez_vlc_android/uploader.py`

```python
from __future__ import annotations

from dataclasses import dataclass
from pathlib import Path
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen

from .config import AppConfig, file_size, normalize_base_url
from .errors import UploadError
from .integrity import sha256_file
from .multipart import build_multipart_body
from .probe import probe_target


@dataclass(slots=True)
class UploadResult:
    base_url: str
    upload_url: str
    filename: str
    size: int
    sha256: str
    status_code: int
    response_text: str


def check_file(path: str | Path) -> None:
    p = Path(path)
    if not p.exists():
        raise UploadError(f"file not found: {p}")
    if not p.is_file():
        raise UploadError(f"not a file: {p}")


def prepare_upload(config: AppConfig, file_path: str | Path) -> dict:
    check_file(file_path)
    path = Path(file_path)
    digest = sha256_file(path)
    body, boundary = build_multipart_body("file", path)
    return {
        "base_url": normalize_base_url(config.base_url),
        "filename": path.name,
        "size": file_size(path),
        "sha256": digest,
        "boundary": boundary,
        "body": body,
    }


def upload_file(config: AppConfig, file_path: str | Path) -> UploadResult:
    base_url = normalize_base_url(config.base_url)
    check_file(file_path)
    path = Path(file_path)

    probe = probe_target(base_url, timeout=config.timeout, user_agent=config.user_agent)
    upload_url = probe.upload_url or f"{base_url}/upload"

    body, boundary = build_multipart_body("file", path)
    req = Request(
        upload_url,
        data=body,
        method="POST",
        headers={
            "User-Agent": config.user_agent,
            "Content-Type": f"multipart/form-data; boundary={boundary}",
            "Content-Length": str(len(body)),
        },
    )

    try:
        with urlopen(req, timeout=config.timeout) as resp:
            response_text = resp.read().decode("utf-8", errors="replace")
            return UploadResult(
                base_url=base_url,
                upload_url=upload_url,
                filename=path.name,
                size=file_size(path),
                sha256=sha256_file(path),
                status_code=resp.status,
                response_text=response_text,
            )
    except HTTPError as e:
        text = e.read().decode("utf-8", errors="replace") if hasattr(e, "read") else str(e)
        raise UploadError(f"upload failed with HTTP {e.code}: {text}") from e
    except URLError as e:
        raise UploadError(f"upload failed: {e}") from e
```

### `src/ez_vlc_android/cli.py`

```python
from __future__ import annotations

import argparse
from pathlib import Path

from .config import AppConfig
from .errors import EZVLCError
from .probe import probe_target
from .uploader import check_file, prepare_upload, upload_file


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="ez-vlc-android")
    sub = parser.add_subparsers(dest="command", required=True)

    p_probe = sub.add_parser("probe", help="Probe a target")
    p_probe.add_argument("base_url")

    p_check = sub.add_parser("check", help="Check a target and file")
    p_check.add_argument("base_url")
    p_check.add_argument("file")

    p_upload = sub.add_parser("upload", help="Upload a file")
    p_upload.add_argument("base_url")
    p_upload.add_argument("file")

    return parser


def main(argv: list[str] | None = None) -> int:
    parser = build_parser()
    args = parser.parse_args(argv)

    try:
        if args.command == "probe":
            result = probe_target(args.base_url)
            print(f"base_url: {result.base_url}")
            print(f"upload_url: {result.upload_url}")
            print(f"confidence: {result.confidence}")
            for item in result.evidence:
                print(f"evidence: {item}")
            return 0

        config = AppConfig(base_url=args.base_url)

        if args.command == "check":
            check_file(Path(args.file))
            result = prepare_upload(config, Path(args.file))
            print(f"base_url: {result['base_url']}")
            print(f"filename: {result['filename']}")
            print(f"size: {result['size']}")
            print(f"sha256: {result['sha256']}")
            print(f"boundary: {result['boundary']}")
            return 0

        if args.command == "upload":
            result = upload_file(config, Path(args.file))
            print(f"base_url: {result.base_url}")
            print(f"upload_url: {result.upload_url}")
            print(f"filename: {result.filename}")
            print(f"size: {result.size}")
            print(f"sha256: {result.sha256}")
            print(f"status_code: {result.status_code}")
            print(result.response_text)
            return 0

        return 1
    except EZVLCError as e:
        print(f"error: {e}")
        return 2
```

### `src/ez_vlc_android/__init__.py`

```python
__version__ = "0.1.0"
```

### `src/ez_vlc_android/__main__.py`

```python
from .cli import main

if __name__ == "__main__":
    raise SystemExit(main())
```

---

## Tests

### `tests/support.py`

```python
from __future__ import annotations

from dataclasses import dataclass
from http.server import BaseHTTPRequestHandler, HTTPServer
from threading import Thread


@dataclass
class MockResponse:
    status: int = 200
    body: bytes = b"OK"
    content_type: str = "text/html"


class MockVlcServer:
    def __init__(self, response: MockResponse | None = None):
        self.response = response or MockResponse()
        self.server = None
        self.thread = None
        self.port = None

    def __enter__(self):
        response = self.response

        class Handler(BaseHTTPRequestHandler):
            def do_GET(self):
                self.send_response(response.status)
                self.send_header("Content-Type", response.content_type)
                self.end_headers()
                self.wfile.write(response.body)

            def do_POST(self):
                self.send_response(response.status)
                self.send_header("Content-Type", response.content_type)
                self.end_headers()
                self.wfile.write(response.body)

            def log_message(self, format, *args):
                return

        self.server = HTTPServer(("127.0.0.1", 0), Handler)
        self.port = self.server.server_address[1]
        self.thread = Thread(target=self.server.serve_forever, daemon=True)
        self.thread.start()
        return self

    def __exit__(self, exc_type, exc, tb):
        if self.server:
            self.server.shutdown()
            self.server.server_close()
        if self.thread:
            self.thread.join(timeout=1)

    @property
    def url(self) -> str:
        return f"http://127.0.0.1:{self.port}"
```

### `tests/test_config.py`

```python
from ez_vlc_android.config import AppConfig, normalize_base_url


def test_normalize_base_url():
    assert normalize_base_url("http://x/") == "http://x"


def test_appconfig_defaults():
    cfg = AppConfig(base_url="http://x")
    assert cfg.timeout == 10.0
    assert cfg.retries == 1
```

### `tests/test_integrity.py`

```python
from ez_vlc_android.integrity import sha256_file


def test_sha256_file(tmp_path):
    f = tmp_path / "a.txt"
    f.write_text("abc")
    assert sha256_file(f) == "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad"
```

### `tests/test_multipart.py`

```python
from pathlib import Path

from ez_vlc_android.multipart import build_multipart_body, guess_mime_type


def test_guess_mime_type():
    assert guess_mime_type("video.mp4") == "video/mp4"


def test_build_multipart_body(tmp_path: Path):
    f = tmp_path / "a.txt"
    f.write_text("hello")
    body, boundary = build_multipart_body("file", f)
    assert boundary.startswith("----ezvlc")
    assert b"hello" in body
    assert b'Content-Disposition: form-data; name="file"; filename="a.txt"' in body
```

### `tests/test_probe.py`

```python
from ez_vlc_android.probe import probe_target
from tests.support import MockResponse, MockVlcServer


def test_probe_target():
    with MockVlcServer(MockResponse(body=b"""
        <html>
          <body>
            <form action="/upload" method="post">
              <input type="file" name="file">
            </form>
          </body>
        </html>
    """)) as server:
        result = probe_target(server.url)
        assert result.base_url == server.url
        assert result.upload_url.endswith("/upload")
        assert result.confidence in {"high", "medium"}
```

### `tests/test_uploader.py`

```python
from pathlib import Path

from ez_vlc_android.config import AppConfig
from ez_vlc_android.uploader import check_file, prepare_upload


def test_check_file(tmp_path: Path):
    f = tmp_path / "movie.mp4"
    f.write_bytes(b"data")
    check_file(f)


def test_prepare_upload(tmp_path: Path):
    f = tmp_path / "movie.mp4"
    f.write_bytes(b"data")
    cfg = AppConfig(base_url="http://example.com")
    result = prepare_upload(cfg, f)
    assert result["filename"] == "movie.mp4"
    assert result["size"] == 4
    assert "sha256" in result
    assert "body" in result
```

### `tests/test_cli.py`

```python
from ez_vlc_android.cli import main
from tests.support import MockResponse, MockVlcServer


def test_cli_probe(capsys):
    with MockVlcServer(MockResponse(body=b"<html><form action='/upload'><input type='file'></form></html>")) as server:
        rc = main(["probe", server.url])
        assert rc == 0
        out = capsys.readouterr().out
        assert "upload_url:" in out


def test_cli_check(tmp_path, capsys):
    f = tmp_path / "a.txt"
    f.write_text("hello")
    rc = main(["check", "http://example.com", str(f)])
    assert rc == 0
    out = capsys.readouterr().out
    assert "sha256:" in out
```

### `tests/__init__.py`

```python
# test package
```

---

## Docs

### `docs/PROBING.md`

```md
# Probing

The probe step:

1. fetches the base page
2. looks for forms
3. checks for file inputs
4. prefers actions containing `upload`
5. falls back to `/upload`

This keeps probing simple and safe.
```

---

## Packaging files

### `.gitignore`

```gitignore
__pycache__/
.pytest_cache/
.ruff_cache/
.mypy_cache/
.venv/
dist/
build/
*.egg-info/
.coverage
htmlcov/
.env
```

### `.editorconfig`

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 4

[*.md]
trim_trailing_whitespace = false
```

### `MANIFEST.in`

```text
include README.md
include LICENSE
recursive-include docs *.md
recursive-include tests *.py
```

### `LICENSE`

```text
MIT License

Copyright (c) 2026 ez-vlc-android

Permission is hereby granted, free of charge, to any person obtaining a copy
...
```

---

If you want, I can next give you:
- a **single shell script** that creates all these files automatically, or
- a **version with richer HTTP handling** using `requests` instead of stdlib.```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT="ez-vlc-android"

mkdir -p "$ROOT"/{docs,src/ez_vlc_android,tests,.github/workflows}

cat > "$ROOT/pyproject.toml" <<'EOF'
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "ez-vlc-android"
version = "0.1.0"
description = "Probe, check, and upload files to VLC Remote Playback on Android"
readme = "README.md"
requires-python = ">=3.10"
license = { text = "MIT" }
authors = [{ name = "Your Name" }]
dependencies = []

[project.optional-dependencies]
dev = [
  "pytest>=8",
]

[project.scripts]
ez-vlc-android = "ez_vlc_android.cli:main"

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"
EOF

cat > "$ROOT/README.md" <<'EOF'
# ez-vlc-android

Probe, check, and upload files to VLC Remote Playback on Android.

## Install

```bash
pip install -e .
```

## Usage

```bash
ez-vlc-android probe http://192.168.1.10:8080
ez-vlc-android check http://192.168.1.10:8080 /path/to/file.mp4
ez-vlc-android upload http://192.168.1.10:8080 /path/to/file.mp4
```

## Development

```bash
python -m pip install -e .[dev]
pytest
```
EOF

cat > "$ROOT/LICENSE" <<'EOF'
MIT License

Copyright (c) 2026 ez-vlc-android

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction...
EOF

cat > "$ROOT/.gitignore" <<'EOF'
__pycache__/
.pytest_cache/
.ruff_cache/
.mypy_cache/
.venv/
dist/
build/
*.egg-info/
.coverage
htmlcov/
.env
EOF

cat > "$ROOT/.editorconfig" <<'EOF'
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 4

[*.md]
trim_trailing_whitespace = false
EOF

cat > "$ROOT/MANIFEST.in" <<'EOF'
include README.md
include LICENSE
recursive-include docs *.md
recursive-include tests *.py
EOF

cat > "$ROOT/docs/PROBING.md" <<'EOF'
# Probing

The probe step:

1. fetches the base page
2. looks for forms
3. checks for file inputs
4. prefers actions containing `upload`
5. falls back to `/upload`

This keeps probing simple and safe.
EOF

cat > "$ROOT/src/ez_vlc_android/__init__.py" <<'EOF'
__version__ = "0.1.0"
EOF

cat > "$ROOT/src/ez_vlc_android/__main__.py" <<'EOF'
from .cli import main

if __name__ == "__main__":
    raise SystemExit(main())
EOF

cat > "$ROOT/src/ez_vlc_android/errors.py" <<'EOF'
class EZVLCError(Exception):
    pass


class ConfigError(EZVLCError):
    pass


class ProbeError(EZVLCError):
    pass


class UploadError(EZVLCError):
    pass


class IntegrityError(EZVLCError):
    pass
EOF

cat > "$ROOT/src/ez_vlc_android/config.py" <<'EOF'
from dataclasses import dataclass
from pathlib import Path


@dataclass(slots=True)
class AppConfig:
    base_url: str
    timeout: float = 10.0
    retries: int = 1
    chunk_size: int = 1024 * 1024
    user_agent: str = "ez-vlc-android/0.1.0"


def normalize_base_url(url: str) -> str:
    return url.rstrip("/")


def file_size(path: str | Path) -> int:
    return Path(path).stat().st_size
EOF

cat > "$ROOT/src/ez_vlc_android/integrity.py" <<'EOF'
from __future__ import annotations

import hashlib
from pathlib import Path


def sha256_file(path: str | Path, chunk_size: int = 1024 * 1024) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            h.update(chunk)
    return h.hexdigest()
EOF

cat > "$ROOT/src/ez_vlc_android/multipart.py" <<'EOF'
from __future__ import annotations

import mimetypes
import uuid
from pathlib import Path


def guess_mime_type(filename: str) -> str:
    mime, _ = mimetypes.guess_type(filename)
    return mime or "application/octet-stream"


def build_multipart_body(field_name: str, file_path: str | Path) -> tuple[bytes, str]:
    path = Path(file_path)
    boundary = f"----ezvlc{uuid.uuid4().hex}"
    mime = guess_mime_type(path.name)
    content = path.read_bytes()

    body = (
        f"--{boundary}\r\n"
        f'Content-Disposition: form-data; name="{field_name}"; filename="{path.name}"\r\n'
        f"Content-Type: {mime}\r\n\r\n"
    ).encode("utf-8") + content + f"\r\n--{boundary}--\r\n".encode("utf-8")

    return body, boundary
EOF

cat > "$ROOT/src/ez_vlc_android/probe.py" <<'EOF'
from __future__ import annotations

from dataclasses import dataclass
from html.parser import HTMLParser
from urllib.parse import urljoin
from urllib.request import Request, urlopen

from .errors import ProbeError


@dataclass(slots=True)
class ProbeResult:
    base_url: str
    upload_url: str | None
    confidence: str
    evidence: list[str]


class _UploadHtmlParser(HTMLParser):
    def __init__(self) -> None:
        super().__init__()
        self.forms: list[dict[str, str | None]] = []
        self.file_inputs = 0
        self.script_srcs: list[str] = []
        self.uploadish_actions: list[str] = []

    def handle_starttag(self, tag, attrs):
        attrs = dict(attrs)
        if tag == "form":
            action = attrs.get("action")
            method = attrs.get("method")
            self.forms.append({"action": action, "method": method})
            if action and "upload" in action.lower():
                self.uploadish_actions.append(action)
        elif tag == "input" and attrs.get("type", "").lower() == "file":
            self.file_inputs += 1
        elif tag == "script" and attrs.get("src"):
            self.script_srcs.append(attrs["src"])


def _fetch(url: str, timeout: float = 10.0, user_agent: str = "ez-vlc-android/0.1.0") -> str:
    req = Request(url, headers={"User-Agent": user_agent})
    with urlopen(req, timeout=timeout) as resp:
        return resp.read().decode("utf-8", errors="replace")


def probe_target(base_url: str, timeout: float = 10.0, user_agent: str = "ez-vlc-android/0.1.0") -> ProbeResult:
    base_url = base_url.rstrip("/")
    evidence: list[str] = []
    upload_url = None
    confidence = "low"

    try:
        html = _fetch(base_url, timeout=timeout, user_agent=user_agent)
    except Exception as e:
        raise ProbeError(f"failed to probe {base_url}: {e}") from e

    parser = _UploadHtmlParser()
    parser.feed(html)

    if parser.file_inputs:
        evidence.append(f"found {parser.file_inputs} file input(s)")
        confidence = "medium"

    for form in parser.forms:
        action = form.get("action")
        if action:
            resolved = urljoin(base_url + "/", action)
            if "upload" in resolved.lower():
                upload_url = resolved
                evidence.append(f"upload-like form action: {action}")
                confidence = "high"
                break

    if upload_url is None and parser.uploadish_actions:
        action = parser.uploadish_actions[0]
        upload_url = urljoin(base_url + "/", action)
        evidence.append(f"fallback upload action: {action}")
        confidence = "medium"

    if upload_url is None:
        upload_url = urljoin(base_url + "/", "upload")
        evidence.append("defaulted to /upload")
        confidence = "low"

    return ProbeResult(
        base_url=base_url,
        upload_url=upload_url,
        confidence=confidence,
        evidence=evidence,
    )
EOF

cat > "$ROOT/src/ez_vlc_android/uploader.py" <<'EOF'
from __future__ import annotations

from dataclasses import dataclass
from pathlib import Path
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen

from .config import AppConfig, file_size, normalize_base_url
from .errors import UploadError
from .integrity import sha256_file
from .multipart import build_multipart_body
from .probe import probe_target


@dataclass(slots=True)
class UploadResult:
    base_url: str
    upload_url: str
    filename: str
    size: int
    sha256: str
    status_code: int
    response_text: str


def check_file(path: str | Path) -> None:
    p = Path(path)
    if not p.exists():
        raise UploadError(f"file not found: {p}")
    if not p.is_file():
        raise UploadError(f"not a file: {p}")


def prepare_upload(config: AppConfig, file_path: str | Path) -> dict:
    check_file(file_path)
    path = Path(file_path)
    digest = sha256_file(path)
    body, boundary = build_multipart_body("file", path)
    return {
        "base_url": normalize_base_url(config.base_url),
        "filename": path.name,
        "size": file_size(path),
        "sha256": digest,
        "boundary": boundary,
        "body": body,
    }


def upload_file(config: AppConfig, file_path: str | Path) -> UploadResult:
    base_url = normalize_base_url(config.base_url)
    check_file(file_path)
    path = Path(file_path)

    probe = probe_target(base_url, timeout=config.timeout, user_agent=config.user_agent)
    upload_url = probe.upload_url or f"{base_url}/upload"

    body, boundary = build_multipart_body("file", path)
    req = Request(
        upload_url,
        data=body,
        method="POST",
        headers={
            "User-Agent": config.user_agent,
            "Content-Type": f"multipart/form-data; boundary={boundary}",
            "Content-Length": str(len(body)),
        },
    )

    try:
        with urlopen(req, timeout=config.timeout) as resp:
            response_text = resp.read().decode("utf-8", errors="replace")
            return UploadResult(
                base_url=base_url,
                upload_url=upload_url,
                filename=path.name,
                size=file_size(path),
                sha256=sha256_file(path),
                status_code=resp.status,
                response_text=response_text,
            )
    except HTTPError as e:
        text = e.read().decode("utf-8", errors="replace") if hasattr(e, "read") else str(e)
        raise UploadError(f"upload failed with HTTP {e.code}: {text}") from e
    except URLError as e:
        raise UploadError(f"upload failed: {e}") from e
EOF

cat > "$ROOT/src/ez_vlc_android/cli.py" <<'EOF'
from __future__ import annotations

import argparse
from pathlib import Path

from .config import AppConfig
from .errors import EZVLCError
from .probe import probe_target
from .uploader import check_file, prepare_upload, upload_file


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="ez-vlc-android")
    sub = parser.add_subparsers(dest="command", required=True)

    p_probe = sub.add_parser("probe", help="Probe a target")
    p_probe.add_argument("base_url")

    p_check = sub.add_parser("check", help="Check a target and file")
    p_check.add_argument("base_url")
    p_check.add_argument("file")

    p_upload = sub.add_parser("upload", help="Upload a file")
    p_upload.add_argument("base_url")
    p_upload.add_argument("file")

    return parser


def main(argv: list[str] | None = None) -> int:
    parser = build_parser()
    args = parser.parse_args(argv)

    try:
        if args.command == "probe":
            result = probe_target(args.base_url)
            print(f"base_url: {result.base_url}")
            print(f"upload_url: {result.upload_url}")
            print(f"confidence: {result.confidence}")
            for item in result.evidence:
                print(f"evidence: {item}")
            return 0

        config = AppConfig(base_url=args.base_url)

        if args.command == "check":
            check_file(Path(args.file))
            result = prepare_upload(config, Path(args.file))
            print(f"base_url: {result['base_url']}")
            print(f"filename: {result['filename']}")
            print(f"size: {result['size']}")
            print(f"sha256: {result['sha256']}")
            print(f"boundary: {result['boundary']}")
            return 0

        if args.command == "upload":
            result = upload_file(config, Path(args.file))
            print(f"base_url: {result.base_url}")
            print(f"upload_url: {result.upload_url}")
            print(f"filename: {result.filename}")
            print(f"size: {result.size}")
            print(f"sha256: {result.sha256}")
            print(f"status_code: {result.status_code}")
            print(result.response_text)
            return 0

        return 1
    except EZVLCError as e:
        print(f"error: {e}")
        return 2
EOF

cat > "$ROOT/tests/__init__.py" <<'EOF'
# test package
EOF

cat > "$ROOT/tests/support.py" <<'EOF'
from __future__ import annotations

from dataclasses import dataclass
from http.server import BaseHTTPRequestHandler, HTTPServer
from threading import Thread


@dataclass
class MockResponse:
    status: int = 200
    body: bytes = b"OK"
    content_type: str = "text/html"


class MockVlcServer:
    def __init__(self, response: MockResponse | None = None):
        self.response = response or MockResponse()
        self.server = None
        self.thread = None
        self.port = None

    def __enter__(self):
        response = self.response

        class Handler(BaseHTTPRequestHandler):
            def do_GET(self):
                self.send_response(response.status)
                self.send_header("Content-Type", response.content_type)
                self.end_headers()
                self.wfile.write(response.body)

            def do_POST(self):
                self.send_response(response.status)
                self.send_header("Content-Type", response.content_type)
                self.end_headers()
                self.wfile.write(response.body)

            def log_message(self, format, *args):
                return

        self.server = HTTPServer(("127.0.0.1", 0), Handler)
        self.port = self.server.server_address[1]
        self.thread = Thread(target=self.server.serve_forever, daemon=True)
        self.thread.start()
        return self

    def __exit__(self, exc_type, exc, tb):
        if self.server:
            self.server.shutdown()
            self.server.server_close()
        if self.thread:
            self.thread.join(timeout=1)

    @property
    def url(self) -> str:
        return f"http://127.0.0.1:{self.port}"
EOF

cat > "$ROOT/tests/test_config.py" <<'EOF'
from ez_vlc_android.config import AppConfig, normalize_base_url


def test_normalize_base_url():
    assert normalize_base_url("http://x/") == "http://x"


def test_appconfig_defaults():
    cfg = AppConfig(base_url="http://x")
    assert cfg.timeout == 10.0
    assert cfg.retries == 1
EOF

cat > "$ROOT/tests/test_integrity.py" <<'EOF'
from ez_vlc_android.integrity import sha256_file


def test_sha256_file(tmp_path):
    f = tmp_path / "a.txt"
    f.write_text("abc")
    assert sha256_file(f) == "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad"
EOF

cat > "$ROOT/tests/test_multipart.py" <<'EOF'
from pathlib import Path

from ez_vlc_android.multipart import build_multipart_body, guess_mime_type


def test_guess_mime_type():
    assert guess_mime_type("video.mp4") == "video/mp4"


def test_build_multipart_body(tmp_path: Path):
    f = tmp_path / "a.txt"
    f.write_text("hello")
    body, boundary = build_multipart_body("file", f)
    assert boundary.startswith("----ezvlc")
    assert b"hello" in body
    assert b'Content-Disposition: form-data; name="file"; filename="a.txt"' in body
EOF

cat > "$ROOT/tests/test_probe.py" <<'EOF'
from ez_vlc_android.probe import probe_target
from tests.support import MockResponse, MockVlcServer


def test_probe_target():
    with MockVlcServer(MockResponse(body=b"""
        <html>
          <body>
            <form action="/upload" method="post">
              <input type="file" name="file">
            </form>
          </body>
        </html>
    """)) as server:
        result = probe_target(server.url)
        assert result.base_url == server.url
        assert result.upload_url.endswith("/upload")
        assert result.confidence in {"high", "medium"}
EOF

cat > "$ROOT/tests/test_uploader.py" <<'EOF'
from pathlib import Path

from ez_vlc_android.config import AppConfig
from ez_vlc_android.uploader import check_file, prepare_upload


def test_check_file(tmp_path: Path):
    f = tmp_path / "movie.mp4"
    f.write_bytes(b"data")
    check_file(f)


def test_prepare_upload(tmp_path: Path):
    f = tmp_path / "movie.mp4"
    f.write_bytes(b"data")
    cfg = AppConfig(base_url="http://example.com")
    result = prepare_upload(cfg, f)
    assert result["filename"] == "movie.mp4"
    assert result["size"] == 4
    assert "sha256" in result
    assert "body" in result
EOF

cat > "$ROOT/tests/test_cli.py" <<'EOF'
from ez_vlc_android.cli import main
from tests.support import MockResponse, MockVlcServer


def test_cli_probe(capsys):
    with MockVlcServer(MockResponse(body=b"<html><form action='/upload'><input type='file'></form></html>")) as server:
        rc = main(["probe", server.url])
        assert rc == 0
        out = capsys.readouterr().out
        assert "upload_url:" in out


def test_cli_check(tmp_path, capsys):
    f = tmp_path / "a.txt"
    f.write_text("hello")
    rc = main(["check", "http://example.com", str(f)])
    assert rc == 0
    out = capsys.readouterr().out
    assert "sha256:" in out
EOF

cat > "$ROOT/.github/workflows/ci.yml" <<'EOF'
name: ci

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.10"
      - run: python -m pip install --upgrade pip
      - run: pip install -e .[dev]
      - run: pytest
EOF

echo "Created project in ./$ROOT"
```
