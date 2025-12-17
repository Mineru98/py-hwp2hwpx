# Java to Python + uv 마이그레이션 기술 리서치

## 1. 현재 프로젝트 분석

### 1.1 프로젝트 개요
- **이름**: hwp2hwpx
- **목적**: HWP(한글) 파일을 HWPX(한글 압축 XML) 파일로 변환
- **언어**: Java 7
- **빌드 도구**: Maven
- **라이선스**: Apache License 2.0

### 1.2 프로젝트 규모
| 항목 | 수치 |
|------|------|
| 총 코드 라인 | ~7,940 줄 |
| 메인 소스 코드 | ~6,642 줄 |
| 클래스 수 (Main) | 66개 |
| 테스트 클래스 수 | 39개 |

### 1.3 핵심 의존성
```xml
<dependency>
    <groupId>kr.dogfoot</groupId>
    <artifactId>hwplib</artifactId>
    <version>1.1.10</version>
</dependency>
<dependency>
    <groupId>kr.dogfoot</groupId>
    <artifactId>hwpxlib</artifactId>
    <version>1.0.8</version>
</dependency>
```

- **hwplib**: HWP 바이너리 파일 파싱 라이브러리
- **hwpxlib**: HWPX XML 파일 생성/쓰기 라이브러리

### 1.4 주요 패키지 구조
```
kr.dogfoot.hwp2hwpx/
├── Hwp2Hwpx.java           # 메인 변환 로직
├── Parameter.java          # 파라미터/상태 관리
├── Converter.java          # 기본 변환기
├── ForContentHPFFile.java  # HPF 파일 처리
├── util/
│   ├── HWPUtil.java        # HWP 유틸리티
│   └── ValueConvertor.java # 값 변환 (544줄)
├── header/                 # 헤더 처리 (14개 파일)
└── section/                # 섹션 처리 (46개 파일)
    └── object/             # 그래픽/폼 객체
        └── gso/            # 그래픽 셰이프 객체
```

---

## 2. Python HWP/HWPX 라이브러리 분석

### 2.1 HWP 파일 읽기 라이브러리

#### Option A: libhwp (Rust 기반 Python 바인딩) ⭐ 추천
- **GitHub**: https://github.com/hahnlee/hwp-rs
- **PyPI**: https://pypi.org/project/libhwp/
- **라이선스**: Apache-2.0
- **Python 버전**: >=3.7
- **설치**: `pip install libhwp`

**장점**:
- Rust로 작성되어 빠른 성능
- 크로스플랫폼 지원 (Windows, macOS, Linux)
- PyO3 기반 안정적인 Python 바인딩
- 활발한 개발 (2022년 릴리스)

**기능**:
```python
from libhwp import HWPReader

hwp = HWPReader('<파일 경로>')
for paragraph in hwp.find_all('paragraph'):
    print(paragraph.text)

for table in hwp.find_all('table'):
    for cell in table.cells:
        for para in cell.paragraphs:
            print(para.text)

# 바이너리 데이터 추출
for data in hwp.bin_data:
    print(data.name, data.data)
```

#### Option B: pyhwp (Pure Python)
- **GitHub**: https://github.com/mete0r/pyhwp
- **PyPI**: https://pypi.org/project/pyhwp/
- **라이선스**: AGPL-3.0 (주의!)
- **Python 버전**: 2.7, 3.5-3.8

**장점**:
- 순수 Python 구현
- ODT/TXT 변환 실험적 지원

**단점**:
- AGPL 라이선스 (상용 제품 사용 시 주의)
- 오래된 Python 버전 지원
- 71개 미해결 이슈

**의존성**: cryptography, lxml, olefile

### 2.2 HWPX 파일 처리

#### HWPX 파일 구조
HWPX는 **ZIP으로 압축된 XML 파일 컬렉션**입니다:
```
example.hwpx (ZIP 파일)
├── META-INF/
│   ├── container.xml
│   └── manifest.xml
├── Contents/
│   ├── content.hpf
│   ├── header.xml
│   ├── section0.xml
│   ├── section1.xml
│   └── ...
├── Preview/
│   └── PrvText.txt
├── version.xml
└── settings.xml
```

#### Python 표준 라이브러리로 처리 가능
```python
import zipfile
import xml.etree.ElementTree as ET
from pathlib import Path

# HWPX 읽기
with zipfile.ZipFile('document.hwpx', 'r') as zf:
    # XML 파싱
    content = zf.read('Contents/header.xml')
    root = ET.fromstring(content)

# HWPX 쓰기
with zipfile.ZipFile('output.hwpx', 'w', zipfile.ZIP_DEFLATED) as zf:
    zf.writestr('version.xml', xml_content)
    zf.writestr('Contents/header.xml', header_content)
```

#### 고급 XML 처리: lxml 권장
```python
from lxml import etree

# 네임스페이스 지원
nsmap = {
    'hp': 'http://www.hancom.co.kr/hwpml/2011/paragraph',
    'hc': 'http://www.hancom.co.kr/hwpml/2011/core'
}

root = etree.Element('{http://www.hancom.co.kr/hwpml/2011/core}hwp', nsmap=nsmap)
```

---

## 3. Python + uv 프로젝트 설정

### 3.1 uv 개요
- **공식 문서**: https://docs.astral.sh/uv/
- **특징**: Rust로 작성된 초고속 Python 패키지 매니저 (pip 대비 10-100x 빠름)
- **기능**: 패키지 관리, 가상환경, Python 버전 관리, 빌드 도구

### 3.2 프로젝트 초기화
```bash
# uv 설치 (아직 설치 안 된 경우)
curl -LsSf https://astral.sh/uv/install.sh | sh

# 새 프로젝트 생성
uv init hwp2hwpx-py
cd hwp2hwpx-py

# 또는 src 레이아웃으로 생성
uv init hwp2hwpx-py --package --build-backend hatchling
```

### 3.3 권장 pyproject.toml 구조
```toml
[project]
name = "hwp2hwpx"
version = "0.1.0"
description = "HWP to HWPX converter"
readme = "README.md"
license = { text = "Apache-2.0" }
requires-python = ">=3.10"
authors = [
    { name = "Your Name", email = "your@email.com" }
]
keywords = ["hwp", "hwpx", "hancom", "document", "converter"]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: Apache Software License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]

dependencies = [
    "libhwp>=0.1.0",
    "lxml>=5.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-cov>=4.0.0",
    "ruff>=0.1.0",
    "mypy>=1.8.0",
]

[project.scripts]
hwp2hwpx = "hwp2hwpx.cli:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.uv]
dev-dependencies = [
    "pytest>=8.0.0",
    "pytest-cov>=4.0.0",
    "ruff>=0.1.0",
]

[tool.ruff]
target-version = "py310"
line-length = 100

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = "test_*.py"
```

### 3.4 프로젝트 구조 (권장)
```
hwp2hwpx-py/
├── pyproject.toml
├── uv.lock
├── README.md
├── LICENSE
├── src/
│   └── hwp2hwpx/
│       ├── __init__.py
│       ├── cli.py              # CLI 인터페이스
│       ├── converter.py        # 메인 변환기
│       ├── parameter.py        # 파라미터/상태 관리
│       ├── util/
│       │   ├── __init__.py
│       │   ├── hwp_util.py
│       │   └── value_converter.py
│       ├── header/
│       │   ├── __init__.py
│       │   ├── header_xml.py
│       │   ├── char_properties.py
│       │   └── ...
│       ├── section/
│       │   ├── __init__.py
│       │   ├── section_xml.py
│       │   └── ...
│       └── hwpx/
│           ├── __init__.py
│           ├── writer.py       # HWPX 파일 쓰기
│           └── models.py       # HWPX 데이터 모델
└── tests/
    ├── __init__.py
    ├── conftest.py
    ├── test_converter.py
    └── fixtures/
        └── *.hwp               # 테스트 HWP 파일
```

### 3.5 주요 uv 명령어
```bash
# 의존성 추가
uv add libhwp lxml

# 개발 의존성 추가
uv add --dev pytest ruff mypy

# 의존성 설치 (lock 파일 기반)
uv sync

# 스크립트 실행
uv run python -m hwp2hwpx input.hwp output.hwpx
uv run pytest

# 빌드
uv build

# 배포
uv publish
```

---

## 4. 마이그레이션 전략

### 4.1 핵심 도전 과제

| 과제 | 설명 | 해결 방안 |
|------|------|-----------|
| hwplib 대체 | Java hwplib의 Python 대체 필요 | libhwp 사용 |
| hwpxlib 대체 | HWPX 파일 생성 라이브러리 없음 | 직접 구현 (zipfile + lxml) |
| 타입 매핑 | Java enum → Python 매핑 | Python Enum 클래스 정의 |
| 빌더 패턴 | Java의 fluent API 변환 | dataclass + 빌더 패턴 |

### 4.2 우선순위별 구현 계획

#### Phase 1: 기초 인프라
1. uv 프로젝트 설정
2. HWPX 파일 구조 모델 정의 (dataclass 기반)
3. HWPX Writer 구현 (zipfile + lxml)
4. 기본 타입/enum 변환 테이블

#### Phase 2: 핵심 변환기
1. HWP 파일 읽기 (libhwp 활용)
2. 헤더 변환 구현
   - 문자 속성
   - 단락 속성
   - 글꼴
   - 테두리/채우기
3. 기본 섹션 변환

#### Phase 3: 고급 요소
1. 표 (Table) 처리
2. 그래픽 객체 (Line, Arc, Rectangle 등)
3. 이미지/OLE 객체
4. 폼 컨트롤

#### Phase 4: 품질 보증
1. 기존 Java 테스트 케이스 마이그레이션
2. 추가 엣지 케이스 테스트
3. 성능 최적화

### 4.3 Java → Python 코드 변환 패턴

#### 예시 1: ValueConvertor 클래스
**Java**:
```java
public static LineType2 lineType2(BorderType hwpType) {
    switch (hwpType) {
        case None: return LineType2.NONE;
        case Solid: return LineType2.SOLID;
        // ...
    }
    return LineType2.NONE;
}
```

**Python**:
```python
from enum import Enum

class BorderType(Enum):
    NONE = "none"
    SOLID = "solid"
    DASH = "dash"
    # ...

class LineType2(Enum):
    NONE = "none"
    SOLID = "solid"
    # ...

BORDER_TO_LINE_TYPE: dict[BorderType, LineType2] = {
    BorderType.NONE: LineType2.NONE,
    BorderType.SOLID: LineType2.SOLID,
    # ...
}

def border_to_line_type(hwp_type: BorderType) -> LineType2:
    return BORDER_TO_LINE_TYPE.get(hwp_type, LineType2.NONE)
```

#### 예시 2: Parameter 클래스
**Java**:
```java
public class Parameter {
    private HWPInfo hwpInfo;
    private HWPXInfo hwpxInfo;
    // ...
}
```

**Python**:
```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Parameter:
    hwp_info: HWPInfo
    hwpx_info: HWPXInfo
    master_page_id_map: dict = field(default_factory=dict)
    bin_data_id_map: dict = field(default_factory=dict)
    field_begin_stack: list = field(default_factory=list)
```

#### 예시 3: Fluent Builder → Python
**Java**:
```java
versionXMLFile
    .targetApplicationAnd(TargetApplicationSort.WordProcessor)
    .osAnd("1")
    .applicationAnd("Hancom Office Hangul");
```

**Python**:
```python
# Option 1: dataclass with factory
version_xml = VersionXML(
    target_application=TargetApplication.WORD_PROCESSOR,
    os="1",
    application="Hancom Office Hangul"
)

# Option 2: Builder pattern
version_xml = (VersionXMLBuilder()
    .target_application(TargetApplication.WORD_PROCESSOR)
    .os("1")
    .application("Hancom Office Hangul")
    .build())
```

---

## 5. HWPX Writer 구현 가이드

### 5.1 기본 구조
```python
import zipfile
from lxml import etree
from pathlib import Path
from dataclasses import dataclass
from typing import Optional

@dataclass
class HWPXFile:
    version: "VersionXML"
    manifest: "ManifestXML"
    container: "ContainerXML"
    content_hpf: "ContentHPF"
    header: "HeaderXML"
    sections: list["SectionXML"]
    settings: Optional["SettingsXML"] = None

class HWPXWriter:
    NAMESPACES = {
        'hp': 'http://www.hancom.co.kr/hwpml/2011/paragraph',
        'hc': 'http://www.hancom.co.kr/hwpml/2011/core',
        'hwp': 'http://www.hancom.co.kr/hwpml/2011/head',
    }

    def __init__(self, hwpx_file: HWPXFile):
        self.hwpx = hwpx_file

    def write(self, output_path: Path) -> None:
        with zipfile.ZipFile(output_path, 'w', zipfile.ZIP_DEFLATED) as zf:
            # version.xml
            zf.writestr('version.xml', self._build_version_xml())

            # META-INF/
            zf.writestr('META-INF/manifest.xml', self._build_manifest_xml())
            zf.writestr('META-INF/container.xml', self._build_container_xml())

            # Contents/
            zf.writestr('Contents/content.hpf', self._build_content_hpf())
            zf.writestr('Contents/header.xml', self._build_header_xml())

            for i, section in enumerate(self.hwpx.sections):
                zf.writestr(f'Contents/section{i}.xml',
                           self._build_section_xml(section))

            # settings.xml
            if self.hwpx.settings:
                zf.writestr('settings.xml', self._build_settings_xml())

    def _build_version_xml(self) -> bytes:
        root = etree.Element('HWPApplicationSetting')
        # ... XML 구성
        return etree.tostring(root, encoding='utf-8', xml_declaration=True)
```

### 5.2 XML 네임스페이스 처리
```python
from lxml import etree

HWPX_NSMAP = {
    None: 'http://www.hancom.co.kr/hwpml/2011/head',
    'hp': 'http://www.hancom.co.kr/hwpml/2011/paragraph',
    'hc': 'http://www.hancom.co.kr/hwpml/2011/core',
}

def create_element(tag: str, nsmap: dict = None, **attrs) -> etree.Element:
    elem = etree.Element(tag, nsmap=nsmap or HWPX_NSMAP)
    for key, value in attrs.items():
        if value is not None:
            elem.set(key, str(value))
    return elem
```

---

## 6. 리스크 및 고려사항

### 6.1 라이선스 이슈
| 라이브러리 | 라이선스 | 상용 사용 |
|-----------|---------|----------|
| libhwp | Apache-2.0 | ✅ 가능 |
| pyhwp | AGPL-3.0 | ⚠️ 주의 필요 |
| lxml | BSD | ✅ 가능 |

### 6.2 기능 완전성
- **libhwp**: 아직 모든 HWP 요소를 지원하지 않을 수 있음
- **직접 구현 필요**: hwpxlib에 해당하는 HWPX Writer 전체 구현

### 6.3 테스트 커버리지
- 기존 39개 테스트 케이스 활용 가능
- `test/` 디렉토리의 HWP 샘플 파일 활용

---

## 7. 참고 자료

### 공식 문서
- [uv 공식 문서](https://docs.astral.sh/uv/)
- [Python Packaging User Guide](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/)

### HWP/HWPX 관련
- [libhwp GitHub](https://github.com/hahnlee/hwp-rs)
- [pyhwp GitHub](https://github.com/mete0r/pyhwp)
- [hwplib GitHub (Java)](https://github.com/neolord0/hwplib)
- [hwpxlib GitHub (Java)](https://github.com/neolord0/hwpxlib)
- [한컴테크 - Python HWPX 파싱](https://tech.hancom.com/python-hwpx-parsing-1/)

### uv 관련
- [uv Deep Dive Guide](https://www.saaspegasus.com/guides/uv-deep-dive/)
- [DataCamp - Python UV Tutorial](https://www.datacamp.com/tutorial/python-uv)
- [Real Python - Managing Projects With uv](https://realpython.com/python-uv/)

---

## 8. 결론 및 권장사항

### 권장 기술 스택
| 영역 | 선택 | 이유 |
|------|------|------|
| 패키지 매니저 | **uv** | 빠른 속도, 현대적 기능 |
| Python 버전 | **3.10+** | 타입 힌팅 강화, 패턴 매칭 |
| HWP 읽기 | **libhwp** | Apache 라이선스, 크로스플랫폼 |
| XML 처리 | **lxml** | 네임스페이스 지원, 성능 |
| HWPX 쓰기 | **직접 구현** | hwpxlib Python 버전 없음 |
| 타입 체크 | **mypy** | 정적 타입 분석 |
| 포매터/린터 | **ruff** | 빠른 속도, 통합 도구 |
| 테스트 | **pytest** | Python 표준 |

### 예상 작업량
- Phase 1 (기초 인프라): 약 500-800 줄
- Phase 2 (핵심 변환기): 약 2,000-3,000 줄
- Phase 3 (고급 요소): 약 2,000-2,500 줄
- Phase 4 (테스트): 약 500-1,000 줄

**총 예상**: 약 5,000-7,000 줄 (Java 원본과 비슷한 규모)

### 다음 단계
1. ✅ 기술 리서치 완료
2. ⬜ uv 프로젝트 초기화
3. ⬜ HWPX 데이터 모델 정의
4. ⬜ HWPX Writer 기본 구현
5. ⬜ libhwp 통합 및 변환기 구현
