# Java to Python + uv 마이그레이션 기술 리서치 (통합본)

## 목차
1. [현재 프로젝트 분석](#1-현재-프로젝트-분석)
2. [HWP 파일 포맷 스펙](#2-hwp-파일-포맷-스펙)
3. [HWPX 파일 포맷 스펙](#3-hwpx-파일-포맷-스펙)
4. [hwplib 라이브러리 구조 분석](#4-hwplib-라이브러리-구조-분석)
5. [hwpxlib 라이브러리 구조 분석](#5-hwpxlib-라이브러리-구조-분석)
6. [Python 구현을 위한 기반 라이브러리](#6-python-구현을-위한-기반-라이브러리)
7. [Python + uv 프로젝트 설정](#7-python--uv-프로젝트-설정)
8. [Python 구현 전략](#8-python-구현-전략)
9. [코드 변환 패턴](#9-코드-변환-패턴)
10. [리스크 및 고려사항](#10-리스크-및-고려사항)
11. [참고 자료](#11-참고-자료)

---

## 1. 현재 프로젝트 분석

### 1.1 프로젝트 개요
| 항목 | 내용 |
|------|------|
| **이름** | hwp2hwpx |
| **목적** | HWP(한글) 파일을 HWPX(한글 압축 XML) 파일로 변환 |
| **언어** | Java 7 |
| **빌드 도구** | Maven |
| **라이선스** | Apache License 2.0 |

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
│   ├── ForHeaderXMLFile.java
│   ├── ForCharProperties.java
│   ├── ForParaProperties.java
│   ├── ForBorderFills.java
│   ├── ForFontfaces.java
│   └── inner/
│       ├── ForFont.java
│       └── ForFillBrush.java
└── section/                # 섹션 처리 (46개 파일)
    ├── ForSectionXMLFileList.java
    ├── ForPara.java
    ├── ForChars.java
    └── object/
        ├── ForTable.java
        ├── ForCell.java
        ├── comm/
        │   ├── ForShapeObject.java
        │   └── ForDrawingObject.java
        └── gso/            # 그래픽 셰이프 객체
            ├── ForLine.java
            ├── ForRectangle.java
            ├── ForPicture.java
            └── form/
                └── ForButton.java
```

---

## 2. HWP 파일 포맷 스펙

### 2.1 공식 스펙 문서
한글과컴퓨터에서 2010년 HWP 파일 포맷 스펙을 공개했습니다:
- **한글문서파일형식_5.0_revision1.2.pdf**: [다운로드](https://cdn.hancom.com/link/docs/한글문서파일형식_5.0_revision1.2.pdf)
- **한글문서파일형식_5.0_revision1.3.pdf**: [다운로드](https://cdn.hancom.com/link/docs/한글문서파일형식_5.0_revision1.3.pdf)

### 2.2 HWP 파일 구조 (CFB 기반)

HWP 파일은 **CFB (Compound File Binary)** 형식을 사용합니다. 이는 Microsoft OLE2 파일 형식과 동일합니다.

```
HWP 파일 (CFB 컨테이너)
├── FileHeader              # 파일 인식 정보 (256 bytes, 비압축)
├── DocInfo                 # 문서 정보 (zlib 압축)
├── BodyText/               # 본문 (Storage)
│   ├── Section0            # 섹션 0 (zlib 압축)
│   ├── Section1            # 섹션 1 (zlib 압축)
│   └── ...
├── BinData/                # 바이너리 데이터 (Storage)
│   ├── BIN0001.jpg         # 이미지 등
│   └── ...
├── PrvText                 # 미리보기 텍스트
├── PrvImage                # 미리보기 이미지
├── DocOptions/             # 문서 옵션 (Storage)
│   └── _LinkDoc
├── Scripts/                # 스크립트 (Storage)
│   ├── DefaultJScript
│   └── JScriptVersion
└── HwpSummaryInformation   # 요약 정보
```

### 2.3 FileHeader 구조 (256 bytes)

```
Offset  Size    Description
------  ----    -----------
0x00    32      Signature ("HWP Document File")
0x20    4       File Version (예: 5.1.0.1)
0x24    4       Properties (비트 플래그)
                - Bit 0: 압축 여부
                - Bit 1: 암호화 여부
                - Bit 2: 배포용 문서
                - Bit 3: 스크립트 저장
                - Bit 4: DRM 보안
                - Bit 5: XML 템플릿 저장
                - Bit 6: 문서 이력 관리
                - Bit 7: 전자 서명
                - Bit 8: 공인 인증서 암호화
                - Bit 9: 전자 서명 예비
                - Bit 10: 공인 인증서 DRM
                - Bit 11: CCL
0x28    216     Reserved
0x100   End
```

### 2.4 레코드 구조

DocInfo와 BodyText 스트림은 **레코드(Record)** 형식으로 저장됩니다:

```
레코드 헤더 (4 bytes)
┌─────────────────────────────────────────────────────────────┐
│ Tag ID (10 bits) │ Level (10 bits) │ Size (12 bits)        │
└─────────────────────────────────────────────────────────────┘

- Tag ID: 레코드 종류 (HWPTAG_* 상수)
- Level: 레코드 깊이 (트리 구조용)
- Size: 데이터 크기 (4095 이하)
        4095인 경우 추가 4바이트로 실제 크기 지정
```

### 2.5 주요 Tag ID 목록

```python
# DocInfo 태그 (0x10 ~ 0x7F)
HWPTAG_DOCUMENT_PROPERTIES  = 0x10  # 문서 속성
HWPTAG_ID_MAPPINGS          = 0x11  # ID 매핑
HWPTAG_BIN_DATA             = 0x12  # 바이너리 데이터
HWPTAG_FACE_NAME            = 0x13  # 글꼴 이름
HWPTAG_BORDER_FILL          = 0x14  # 테두리/채우기
HWPTAG_CHAR_SHAPE           = 0x15  # 글자 모양
HWPTAG_TAB_DEF              = 0x16  # 탭 정의
HWPTAG_NUMBERING            = 0x17  # 번호 매기기
HWPTAG_BULLET               = 0x18  # 글머리표
HWPTAG_PARA_SHAPE           = 0x19  # 문단 모양
HWPTAG_STYLE                = 0x1A  # 스타일
HWPTAG_MEMO_SHAPE           = 0x1B  # 메모 모양
HWPTAG_FORBIDDEN_CHAR       = 0x1E  # 금칙 문자

# BodyText 태그 (0x50 ~ 0x7F)
HWPTAG_PARA_HEADER          = 0x42  # 문단 헤더
HWPTAG_PARA_TEXT            = 0x43  # 문단 텍스트
HWPTAG_PARA_CHAR_SHAPE      = 0x44  # 문단 글자 모양
HWPTAG_PARA_LINE_SEG        = 0x45  # 문단 레이아웃
HWPTAG_CTRL_HEADER          = 0x4B  # 컨트롤 헤더
HWPTAG_LIST_HEADER          = 0x4C  # 리스트 헤더
HWPTAG_TABLE                = 0x4D  # 표
HWPTAG_SHAPE_COMPONENT      = 0x4E  # 그리기 객체
```

### 2.6 데이터 타입

```python
# Little Endian 바이트 순서
INT8    = 1 byte signed
INT16   = 2 bytes signed
INT32   = 4 bytes signed
UINT8   = 1 byte unsigned
UINT16  = 2 bytes unsigned
UINT32  = 4 bytes unsigned
HWPUNIT = INT32 (1/7200 inch)
SHWPUNIT = INT16 (1/7200 inch)
COLORREF = UINT32 (0x00BBGGRR)
WCHAR   = 2 bytes (UTF-16LE)
```

### 2.7 압축/암호화

- **압축**: zlib (DEFLATE) 알고리즘 사용
- **암호화**: 지원하지 않음 (별도 처리 필요)

```python
import zlib

def decompress_stream(data: bytes) -> bytes:
    """HWP 스트림 압축 해제 (raw deflate)"""
    return zlib.decompress(data, -15)  # -15 = raw deflate
```

---

## 3. HWPX 파일 포맷 스펙

### 3.1 개요

HWPX는 **OWPML (Open Word-Processor Markup Language)** 기반의 XML 포맷입니다:
- **표준**: KS X 6101 (2011년 제정)
- **형식**: ZIP 압축 + XML
- **기본 포맷 전환**: 2021년 4월 15일부터 한글의 기본 저장 형식

### 3.2 HWPX 파일 구조

```
example.hwpx (ZIP 아카이브)
├── mimetype                    # MIME 타입 식별자
├── version.xml                 # 버전 정보
├── settings.xml                # 설정 정보
├── META-INF/
│   ├── container.xml           # 컨테이너 정보
│   └── manifest.xml            # 매니페스트
├── Contents/
│   ├── content.hpf             # 콘텐츠 패키지 (OPF 형식)
│   ├── header.xml              # 헤더 (스타일, 글꼴 등)
│   ├── section0.xml            # 섹션 0 본문
│   ├── section1.xml            # 섹션 1 본문
│   └── ...
├── BinData/                    # 바이너리 데이터
│   ├── BIN0001.jpg
│   └── ...
└── Preview/
    └── PrvText.txt             # 미리보기 텍스트
```

### 3.3 주요 XML 파일 상세

#### version.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<HWPApplicationSetting xmlns="http://www.hancom.co.kr/hwpml/2011/app">
    <APPLICATIONVERSION Application="Hancom Office Hangul"
                        AppVersion="9, 1, 1, 5656"/>
    <VERSION Major="5" Minor="1" Micro="0" BuildNumber="1"/>
</HWPApplicationSetting>
```

#### Contents/header.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<hh:head xmlns:hh="http://www.hancom.co.kr/hwpml/2011/head"
         xmlns:hc="http://www.hancom.co.kr/hwpml/2011/core">
    <hh:refList>
        <hh:fontfaces>
            <hh:fontface Lang="HANGUL" Count="2">
                <hh:font Id="0" Type="TTF" Name="함초롬돋움"/>
                <hh:font Id="1" Type="TTF" Name="함초롬바탕"/>
            </hh:fontface>
        </hh:fontfaces>
        <hh:charProperties>
            <hh:charPr Id="0">
                <hh:fontRef Hangul="0" Latin="0"/>
                <hh:ratio Hangul="100" Latin="100"/>
            </hh:charPr>
        </hh:charProperties>
        <hh:paraProperties>
            <hh:paraPr Id="0">
                <hh:align Horizontal="JUSTIFY" Vertical="BASELINE"/>
            </hh:paraPr>
        </hh:paraProperties>
    </hh:refList>
</hh:head>
```

#### Contents/section0.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<hs:sec xmlns:hs="http://www.hancom.co.kr/hwpml/2011/section"
        xmlns:hp="http://www.hancom.co.kr/hwpml/2011/paragraph">
    <hp:p ParaShapeIDRef="0" StyleIDRef="0">
        <hp:run CharShapeIDRef="0">
            <hp:t>안녕하세요</hp:t>
        </hp:run>
    </hp:p>
</hs:sec>
```

### 3.4 XML 네임스페이스

```python
HWPX_NAMESPACES = {
    'hh': 'http://www.hancom.co.kr/hwpml/2011/head',      # Header
    'hs': 'http://www.hancom.co.kr/hwpml/2011/section',   # Section
    'hp': 'http://www.hancom.co.kr/hwpml/2011/paragraph', # Paragraph
    'hc': 'http://www.hancom.co.kr/hwpml/2011/core',      # Core
    'ha': 'http://www.hancom.co.kr/hwpml/2011/app',       # Application
    'ooxmlchart': 'http://www.hancom.co.kr/hwpml/2016/HwpUnitChart',
}
```

---

## 4. hwplib 라이브러리 구조 분석

### 4.1 개요
- **GitHub**: https://github.com/neolord0/hwplib
- **라이선스**: Apache-2.0
- **기능**: HWP 파일 읽기/쓰기
- **의존성**: Apache POI (CFB 파싱)

### 4.2 패키지 구조

```
kr.dogfoot.hwplib/
├── object/                     # HWP 문서 객체 모델
│   ├── HWPFile.java           # 최상위 파일 객체
│   ├── fileheader/            # 파일 헤더
│   │   └── FileHeader.java
│   ├── docinfo/               # 문서 정보
│   │   ├── DocInfo.java
│   │   ├── DocumentProperties.java
│   │   ├── FaceName.java
│   │   ├── CharShape.java
│   │   ├── ParaShape.java
│   │   ├── BorderFill.java
│   │   ├── Style.java
│   │   └── charshape/
│   │       ├── EmphasisSort.java
│   │       └── UnderLineSort.java
│   ├── bodytext/              # 본문
│   │   ├── BodyText.java
│   │   ├── Section.java
│   │   ├── paragraph/
│   │   │   ├── Paragraph.java
│   │   │   ├── ParaText.java
│   │   │   └── charshape/
│   │   └── control/           # 컨트롤 객체
│   │       ├── Control.java
│   │       ├── ControlTable.java
│   │       └── gso/           # 그래픽 객체
│   │           ├── ControlRectangle.java
│   │           └── ControlPicture.java
│   ├── bindata/               # 바이너리 데이터
│   │   └── BinData.java
│   └── etc/
│       └── Color4Byte.java
├── reader/                    # HWP 읽기
│   ├── HWPReader.java         # 메인 리더
│   ├── docinfo/
│   └── bodytext/
├── writer/                    # HWP 쓰기
│   └── HWPWriter.java
└── util/                      # 유틸리티
    └── compressor/
        └── Compressor.java
```

### 4.3 주요 클래스 상세

#### HWPFile.java
```java
public class HWPFile {
    private FileHeader fileHeader;      // 파일 헤더
    private DocInfo docInfo;            // 문서 정보
    private BodyText bodyText;          // 본문
    private BinData binData;            // 바이너리 데이터
    private SummaryInformation summary; // 요약 정보

    // Getters and Setters...
}
```

#### DocInfo 주요 구성요소
```java
public class DocInfo {
    private DocumentProperties documentProperties; // 문서 속성
    private ArrayList<FaceName> faceNameList;      // 글꼴 목록
    private ArrayList<BorderFill> borderFillList;  // 테두리/채우기
    private ArrayList<CharShape> charShapeList;    // 글자 모양
    private ArrayList<ParaShape> paraShapeList;    // 문단 모양
    private ArrayList<Style> styleList;            // 스타일
    private ArrayList<Numbering> numberingList;    // 번호 매기기
    private ArrayList<Bullet> bulletList;          // 글머리표
}
```

### 4.4 Python 구현 시 고려사항

hwplib를 Python으로 구현하려면:
1. **olefile**: CFB 파일 읽기/쓰기
2. **zlib**: 스트림 압축/해제
3. **struct**: 바이너리 데이터 파싱
4. **dataclass**: 객체 모델 정의
5. **enum**: 상수 정의

---

## 5. hwpxlib 라이브러리 구조 분석

### 5.1 개요
- **GitHub**: https://github.com/neolord0/hwpxlib
- **라이선스**: Apache-2.0
- **기능**: HWPX 파일 읽기/쓰기
- **의존성**: SAX Parser (XML 처리)

### 5.2 패키지 구조

```
kr.dogfoot.hwpxlib/
├── object/                        # HWPX 문서 객체 모델
│   ├── HWPXFile.java             # 최상위 파일 객체
│   ├── root/
│   │   ├── VersionXMLFile.java   # version.xml
│   │   └── SettingsXMLFile.java  # settings.xml
│   ├── metainf/
│   │   ├── ContainerXMLFile.java # container.xml
│   │   └── ManifestXMLFile.java  # manifest.xml
│   ├── content/
│   │   ├── context_hpf/
│   │   │   └── ContentHPFFile.java
│   │   ├── header_xml/
│   │   │   ├── HeaderXMLFile.java
│   │   │   ├── RefList.java
│   │   │   └── enumtype/
│   │   │       ├── LineType2.java
│   │   │       └── HorizontalAlign2.java
│   │   ├── section_xml/
│   │   │   ├── SectionXMLFile.java
│   │   │   ├── paragraph/
│   │   │   │   ├── Para.java
│   │   │   │   └── Run.java
│   │   │   └── enumtype/
│   │   └── masterpage_xml/
│   └── common/
│       └── ObjectList.java
├── reader/                        # HWPX 읽기
│   └── HWPXReader.java
├── writer/                        # HWPX 쓰기
│   └── HWPXWriter.java
└── util/
    ├── ObjectFinder.java
    └── TextExtractor.java
```

### 5.3 주요 클래스 상세

#### HWPXFile.java
```java
public class HWPXFile {
    private VersionXMLFile versionXMLFile;      // version.xml
    private SettingsXMLFile settingsXMLFile;    // settings.xml
    private ContainerXMLFile containerXMLFile;  // container.xml
    private ManifestXMLFile manifestXMLFile;    // manifest.xml
    private ContentHPFFile contentHPFFile;      // content.hpf
    private HeaderXMLFile headerXMLFile;        // header.xml
    private ObjectList<SectionXMLFile> sectionXMLFileList;  // section*.xml
    private ObjectList<MasterPageXMLFile> masterPageXMLFileList;
    private BinDataFiles binDataFiles;          // BinData/
}
```

### 5.4 Python 구현 시 고려사항

hwpxlib를 Python으로 구현하려면:
1. **zipfile**: ZIP 압축/해제
2. **lxml**: XML 파싱/생성 (네임스페이스 지원)
3. **dataclass**: 객체 모델 정의
4. **enum**: enumtype 변환

---

## 6. Python 구현을 위한 기반 라이브러리

### 6.1 필수 라이브러리

| 라이브러리 | 용도 | 설치 |
|-----------|------|------|
| **olefile** | CFB/OLE2 파일 읽기 | `uv add olefile` |
| **lxml** | XML 파싱/생성 | `uv add lxml` |
| **cryptography** | 암호화 처리 (선택) | `uv add cryptography` |

### 6.2 표준 라이브러리

| 모듈 | 용도 |
|------|------|
| **zipfile** | HWPX ZIP 압축/해제 |
| **zlib** | HWP 스트림 압축/해제 |
| **struct** | 바이너리 데이터 파싱 |
| **enum** | 열거형 정의 |
| **dataclasses** | 데이터 클래스 정의 |
| **typing** | 타입 힌트 |
| **pathlib** | 파일 경로 처리 |

### 6.3 olefile 사용법

```python
import olefile

def read_hwp_file(filepath: str) -> dict:
    """HWP 파일의 스트림들을 읽기"""
    ole = olefile.OleFileIO(filepath)

    streams = {}

    # FileHeader 읽기 (비압축)
    if ole.exists('FileHeader'):
        streams['FileHeader'] = ole.openstream('FileHeader').read()

    # DocInfo 읽기 (압축)
    if ole.exists('DocInfo'):
        streams['DocInfo'] = ole.openstream('DocInfo').read()

    # BodyText 섹션들 읽기
    section_idx = 0
    while ole.exists(f'BodyText/Section{section_idx}'):
        streams[f'Section{section_idx}'] = ole.openstream(
            f'BodyText/Section{section_idx}'
        ).read()
        section_idx += 1

    # BinData 읽기
    if ole.exists('BinData'):
        for entry in ole.listdir():
            if entry[0] == 'BinData' and len(entry) > 1:
                path = '/'.join(entry)
                streams[path] = ole.openstream(path).read()

    ole.close()
    return streams
```

### 6.4 struct를 이용한 바이너리 파싱

```python
import struct
from dataclasses import dataclass
from enum import IntEnum

class HWPTag(IntEnum):
    """HWP 태그 ID"""
    DOCUMENT_PROPERTIES = 0x10
    ID_MAPPINGS = 0x11
    BIN_DATA = 0x12
    FACE_NAME = 0x13
    BORDER_FILL = 0x14
    CHAR_SHAPE = 0x15
    TAB_DEF = 0x16
    NUMBERING = 0x17
    BULLET = 0x18
    PARA_SHAPE = 0x19
    STYLE = 0x1A

@dataclass
class RecordHeader:
    """HWP 레코드 헤더"""
    tag_id: int
    level: int
    size: int

def parse_record_header(data: bytes, offset: int) -> tuple[RecordHeader, int]:
    """레코드 헤더 파싱"""
    # 4바이트 읽기 (Little Endian)
    header_value = struct.unpack_from('<I', data, offset)[0]

    tag_id = header_value & 0x3FF          # 하위 10비트
    level = (header_value >> 10) & 0x3FF   # 중간 10비트
    size = (header_value >> 20) & 0xFFF    # 상위 12비트

    next_offset = offset + 4

    # 크기가 4095이면 추가 4바이트에서 실제 크기 읽기
    if size == 0xFFF:
        size = struct.unpack_from('<I', data, next_offset)[0]
        next_offset += 4

    return RecordHeader(tag_id, level, size), next_offset

def parse_records(data: bytes) -> list[tuple[RecordHeader, bytes]]:
    """모든 레코드 파싱"""
    records = []
    offset = 0

    while offset < len(data):
        header, new_offset = parse_record_header(data, offset)
        record_data = data[new_offset:new_offset + header.size]
        records.append((header, record_data))
        offset = new_offset + header.size

    return records
```

### 6.5 zlib 압축 해제

```python
import zlib

def decompress_hwp_stream(compressed_data: bytes) -> bytes:
    """HWP 스트림 압축 해제

    HWP는 raw deflate를 사용하므로 wbits=-15 설정
    """
    try:
        # Raw deflate (no header)
        return zlib.decompress(compressed_data, wbits=-15)
    except zlib.error:
        # zlib header 포함된 경우
        return zlib.decompress(compressed_data)

def compress_hwp_stream(data: bytes) -> bytes:
    """HWP 스트림 압축"""
    compress_obj = zlib.compressobj(
        level=zlib.Z_DEFAULT_COMPRESSION,
        method=zlib.DEFLATED,
        wbits=-15  # Raw deflate
    )
    return compress_obj.compress(data) + compress_obj.flush()
```

### 6.6 lxml을 이용한 XML 처리

```python
from lxml import etree
from typing import Optional

# 네임스페이스 정의
NSMAP = {
    'hh': 'http://www.hancom.co.kr/hwpml/2011/head',
    'hs': 'http://www.hancom.co.kr/hwpml/2011/section',
    'hp': 'http://www.hancom.co.kr/hwpml/2011/paragraph',
    'hc': 'http://www.hancom.co.kr/hwpml/2011/core',
}

def create_element(
    tag: str,
    namespace: Optional[str] = None,
    nsmap: dict = None,
    **attrs
) -> etree.Element:
    """XML 요소 생성"""
    if namespace:
        full_tag = f'{{{NSMAP[namespace]}}}{tag}'
    else:
        full_tag = tag

    elem = etree.Element(full_tag, nsmap=nsmap or NSMAP)

    for key, value in attrs.items():
        if value is not None:
            elem.set(key, str(value))

    return elem

def build_header_xml(header_data: dict) -> bytes:
    """header.xml 생성"""
    root = create_element('head', namespace='hh')

    ref_list = etree.SubElement(root, f'{{{NSMAP["hh"]}}}refList')

    # fontfaces
    fontfaces = etree.SubElement(ref_list, f'{{{NSMAP["hh"]}}}fontfaces')
    for lang, fonts in header_data['fonts'].items():
        fontface = etree.SubElement(fontfaces, f'{{{NSMAP["hh"]}}}fontface')
        fontface.set('Lang', lang)
        fontface.set('Count', str(len(fonts)))
        for i, font_name in enumerate(fonts):
            font = etree.SubElement(fontface, f'{{{NSMAP["hh"]}}}font')
            font.set('Id', str(i))
            font.set('Type', 'TTF')
            font.set('Name', font_name)

    return etree.tostring(
        root,
        encoding='utf-8',
        xml_declaration=True,
        pretty_print=True
    )
```

---

## 7. Python + uv 프로젝트 설정

### 7.1 uv 개요
- **공식 문서**: https://docs.astral.sh/uv/
- **특징**: Rust로 작성된 초고속 Python 패키지 매니저 (pip 대비 10-100x 빠름)
- **기능**: 패키지 관리, 가상환경, Python 버전 관리, 빌드 도구

### 7.2 프로젝트 초기화
```bash
# uv 설치
curl -LsSf https://astral.sh/uv/install.sh | sh

# 프로젝트 생성 (src 레이아웃, hatchling 빌드 백엔드)
uv init hwp2hwpx-py --package --build-backend hatchling
cd hwp2hwpx-py
```

### 7.3 권장 pyproject.toml

```toml
[project]
name = "hwp2hwpx"
version = "0.1.0"
description = "HWP to HWPX converter - Pure Python implementation"
readme = "README.md"
license = { text = "Apache-2.0" }
requires-python = ">=3.10"
authors = [
    { name = "Your Name", email = "your@email.com" }
]
keywords = ["hwp", "hwpx", "hancom", "document", "converter", "korean"]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: Apache Software License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
    "Operating System :: OS Independent",
    "Topic :: Office/Business :: Office Suites",
    "Topic :: Text Processing",
]

dependencies = [
    "olefile>=0.47",        # CFB/OLE2 파일 처리
    "lxml>=5.0.0",          # XML 파싱/생성
]

[project.optional-dependencies]
crypto = [
    "cryptography>=42.0.0", # 암호화 파일 처리
]
dev = [
    "pytest>=8.0.0",
    "pytest-cov>=4.0.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
    "types-lxml>=2024.0.0",
]

[project.scripts]
hwp2hwpx = "hwp2hwpx.cli:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/hwp2hwpx"]

[tool.uv]
dev-dependencies = [
    "pytest>=8.0.0",
    "pytest-cov>=4.0.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
]

[tool.ruff]
target-version = "py310"
line-length = 100
src = ["src"]

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B", "C4", "SIM"]

[tool.mypy]
python_version = "3.10"
strict = true
warn_return_any = true

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = "test_*.py"
addopts = "-v --cov=hwp2hwpx --cov-report=term-missing"
```

### 7.4 권장 프로젝트 구조

```
hwp2hwpx-py/
├── pyproject.toml
├── uv.lock
├── README.md
├── LICENSE
├── src/
│   └── hwp2hwpx/
│       ├── __init__.py
│       ├── py.typed                 # PEP 561 마커
│       ├── cli.py                   # CLI 인터페이스
│       ├── converter.py             # 메인 변환기
│       │
│       ├── hwp/                     # HWP 파일 처리 (hwplib 대체)
│       │   ├── __init__.py
│       │   ├── reader.py            # HWP 파일 읽기
│       │   ├── writer.py            # HWP 파일 쓰기
│       │   ├── models/              # 데이터 모델
│       │   │   ├── __init__.py
│       │   │   ├── file.py          # HWPFile
│       │   │   ├── file_header.py   # FileHeader
│       │   │   ├── doc_info.py      # DocInfo
│       │   │   ├── body_text.py     # BodyText, Section
│       │   │   ├── paragraph.py     # Paragraph
│       │   │   ├── control.py       # Control 객체들
│       │   │   └── bin_data.py      # BinData
│       │   ├── enums/               # 열거형
│       │   │   ├── __init__.py
│       │   │   ├── tags.py          # HWP 태그 ID
│       │   │   ├── char_shape.py    # 글자 모양 관련
│       │   │   ├── para_shape.py    # 문단 모양 관련
│       │   │   └── border_fill.py   # 테두리/채우기 관련
│       │   └── parser/              # 파서
│       │       ├── __init__.py
│       │       ├── record.py        # 레코드 파서
│       │       ├── doc_info.py      # DocInfo 파서
│       │       └── body_text.py     # BodyText 파서
│       │
│       ├── hwpx/                    # HWPX 파일 처리 (hwpxlib 대체)
│       │   ├── __init__.py
│       │   ├── reader.py            # HWPX 파일 읽기
│       │   ├── writer.py            # HWPX 파일 쓰기
│       │   ├── models/              # 데이터 모델
│       │   │   ├── __init__.py
│       │   │   ├── file.py          # HWPXFile
│       │   │   ├── version.py       # VersionXML
│       │   │   ├── header.py        # HeaderXML
│       │   │   ├── section.py       # SectionXML
│       │   │   └── settings.py      # SettingsXML
│       │   ├── enums/               # 열거형
│       │   │   ├── __init__.py
│       │   │   ├── line_type.py
│       │   │   ├── align.py
│       │   │   └── number_type.py
│       │   └── builder/             # XML 빌더
│       │       ├── __init__.py
│       │       ├── header.py
│       │       └── section.py
│       │
│       ├── transform/               # HWP → HWPX 변환
│       │   ├── __init__.py
│       │   ├── header.py            # 헤더 변환
│       │   ├── section.py           # 섹션 변환
│       │   ├── paragraph.py         # 문단 변환
│       │   ├── table.py             # 표 변환
│       │   └── shape.py             # 그래픽 객체 변환
│       │
│       └── util/                    # 유틸리티
│           ├── __init__.py
│           ├── compression.py       # 압축/해제
│           ├── binary.py            # 바이너리 처리
│           └── value_converter.py   # 값 변환
│
└── tests/
    ├── __init__.py
    ├── conftest.py                  # pytest 설정
    ├── hwp/
    │   ├── test_reader.py
    │   └── test_parser.py
    ├── hwpx/
    │   ├── test_writer.py
    │   └── test_builder.py
    ├── transform/
    │   └── test_converter.py
    └── fixtures/                    # 테스트 파일
        ├── sample.hwp
        └── expected/
            └── sample.hwpx
```

### 7.5 주요 uv 명령어

```bash
# 의존성 추가
uv add olefile lxml

# 개발 의존성 추가
uv add --dev pytest pytest-cov ruff mypy

# 선택적 의존성 추가
uv add --optional crypto cryptography

# 의존성 설치 (lock 파일 기반)
uv sync

# 개발 의존성 포함 설치
uv sync --all-extras

# 스크립트 실행
uv run python -m hwp2hwpx input.hwp output.hwpx
uv run pytest

# 린트/포맷
uv run ruff check src/
uv run ruff format src/

# 타입 체크
uv run mypy src/

# 빌드
uv build

# 배포
uv publish
```

---

## 8. Python 구현 전략

### 8.1 구현 우선순위

#### Phase 1: 기초 인프라 (예상 1,500줄)
1. 프로젝트 구조 설정
2. HWP 레코드 파서 구현
3. HWP 기본 모델 정의 (FileHeader, DocInfo 구조)
4. HWPX 기본 모델 정의
5. HWPX Writer 기본 구현

#### Phase 2: DocInfo 변환 (예상 2,500줄)
1. 글꼴 (FaceName) 파싱 및 변환
2. 글자 모양 (CharShape) 파싱 및 변환
3. 문단 모양 (ParaShape) 파싱 및 변환
4. 테두리/채우기 (BorderFill) 파싱 및 변환
5. 스타일 (Style) 파싱 및 변환

#### Phase 3: BodyText 변환 (예상 3,000줄)
1. 섹션 (Section) 파싱 및 변환
2. 문단 (Paragraph) 파싱 및 변환
3. 표 (Table) 파싱 및 변환
4. 그래픽 객체 기본 파싱 및 변환

#### Phase 4: 고급 기능 (예상 2,000줄)
1. 이미지/OLE 객체 처리
2. 폼 컨트롤 처리
3. 각주/미주 처리
4. 머리말/꼬리말 처리

#### Phase 5: 품질 보증 (예상 1,000줄)
1. 테스트 케이스 작성
2. 엣지 케이스 처리
3. 성능 최적화
4. 문서화

**총 예상**: 약 10,000줄 (hwplib + hwpxlib + hwp2hwpx 통합)

### 8.2 핵심 클래스 설계

#### HWPFile (hwp/models/file.py)
```python
from dataclasses import dataclass, field
from typing import Optional
from .file_header import FileHeader
from .doc_info import DocInfo
from .body_text import BodyText
from .bin_data import BinData

@dataclass
class HWPFile:
    """HWP 파일 최상위 객체"""
    file_header: FileHeader
    doc_info: DocInfo
    body_text: BodyText
    bin_data: BinData = field(default_factory=BinData)

    @classmethod
    def from_file(cls, filepath: str) -> "HWPFile":
        """파일에서 HWP 읽기"""
        from ..reader import HWPReader
        return HWPReader.read(filepath)

    def to_hwpx(self) -> "HWPXFile":
        """HWPX로 변환"""
        from ...transform import convert_hwp_to_hwpx
        return convert_hwp_to_hwpx(self)
```

#### HWPXFile (hwpx/models/file.py)
```python
from dataclasses import dataclass, field
from typing import Optional
from pathlib import Path
from .version import VersionXML
from .header import HeaderXML
from .section import SectionXML
from .settings import SettingsXML

@dataclass
class HWPXFile:
    """HWPX 파일 최상위 객체"""
    version: VersionXML
    header: HeaderXML
    sections: list[SectionXML] = field(default_factory=list)
    settings: Optional[SettingsXML] = None
    bin_data: dict[str, bytes] = field(default_factory=dict)

    def write(self, filepath: str | Path) -> None:
        """파일로 HWPX 쓰기"""
        from ..writer import HWPXWriter
        HWPXWriter.write(self, filepath)
```

---

## 9. 코드 변환 패턴

### 9.1 Java enum → Python Enum

**Java**:
```java
public enum BorderType {
    None,
    Solid,
    Dash,
    Dot,
    DashDot,
    // ...
}
```

**Python**:
```python
from enum import Enum, auto

class BorderType(Enum):
    """테두리 종류"""
    NONE = auto()
    SOLID = auto()
    DASH = auto()
    DOT = auto()
    DASH_DOT = auto()

    @classmethod
    def from_value(cls, value: int) -> "BorderType":
        """정수값에서 변환"""
        mapping = {
            0: cls.NONE,
            1: cls.SOLID,
            2: cls.DASH,
            3: cls.DOT,
            4: cls.DASH_DOT,
        }
        return mapping.get(value, cls.NONE)
```

### 9.2 Java switch → Python dict 매핑

**Java**:
```java
public static LineType2 lineType2(BorderType hwpType) {
    switch (hwpType) {
        case None: return LineType2.NONE;
        case Solid: return LineType2.SOLID;
        case Dash: return LineType2.DASH;
        // ...
    }
    return LineType2.NONE;
}
```

**Python**:
```python
from typing import Final

BORDER_TO_LINE_TYPE: Final[dict[BorderType, LineType2]] = {
    BorderType.NONE: LineType2.NONE,
    BorderType.SOLID: LineType2.SOLID,
    BorderType.DASH: LineType2.DASH,
    BorderType.DOT: LineType2.DOT,
    BorderType.DASH_DOT: LineType2.DASH_DOT,
}

def border_to_line_type(hwp_type: BorderType) -> LineType2:
    """BorderType을 LineType2로 변환"""
    return BORDER_TO_LINE_TYPE.get(hwp_type, LineType2.NONE)
```

### 9.3 Java Fluent Builder → Python dataclass

**Java**:
```java
versionXMLFile
    .targetApplicationAnd(TargetApplicationSort.WordProcessor)
    .osAnd("1")
    .applicationAnd("Hancom Office Hangul")
    .appVersion("9, 1, 1, 5656");
```

**Python (dataclass)**:
```python
from dataclasses import dataclass
from enum import Enum

class TargetApplication(Enum):
    WORD_PROCESSOR = "WordProcessor"
    SPREADSHEET = "SpreadSheet"

@dataclass
class VersionXML:
    """version.xml 데이터"""
    target_application: TargetApplication = TargetApplication.WORD_PROCESSOR
    os: str = "1"
    application: str = "Hancom Office Hangul"
    app_version: str = "9, 1, 1, 5656"
    major: int = 5
    minor: int = 1
    micro: int = 0
    build_number: int = 1

# 사용
version = VersionXML(
    target_application=TargetApplication.WORD_PROCESSOR,
    os="1",
    application="Hancom Office Hangul",
    app_version="9, 1, 1, 5656"
)
```

### 9.4 Java 바이트 파싱 → Python struct

**Java**:
```java
int tagId = value & 0x3FF;
int level = (value >> 10) & 0x3FF;
int size = (value >> 20) & 0xFFF;
```

**Python**:
```python
import struct

def parse_record_header(data: bytes, offset: int = 0) -> tuple[int, int, int, int]:
    """레코드 헤더 파싱

    Returns:
        (tag_id, level, size, next_offset)
    """
    value = struct.unpack_from('<I', data, offset)[0]

    tag_id = value & 0x3FF
    level = (value >> 10) & 0x3FF
    size = (value >> 20) & 0xFFF

    next_offset = offset + 4

    if size == 0xFFF:
        size = struct.unpack_from('<I', data, next_offset)[0]
        next_offset += 4

    return tag_id, level, size, next_offset
```

### 9.5 Java Color → Python

**Java**:
```java
public static String color(Color4Byte hwpColor) {
    if (hwpColor.getValue() == 4294967295L) return "none";
    return String.format("#%02X%02X%02X",
            hwpColor.getR(),
            hwpColor.getG(),
            hwpColor.getB());
}
```

**Python**:
```python
@dataclass
class Color4Byte:
    """4바이트 색상 (0x00BBGGRR)"""
    value: int

    @property
    def r(self) -> int:
        return self.value & 0xFF

    @property
    def g(self) -> int:
        return (self.value >> 8) & 0xFF

    @property
    def b(self) -> int:
        return (self.value >> 16) & 0xFF

    def to_hex(self) -> str:
        """#RRGGBB 형식 문자열로 변환"""
        if self.value == 0xFFFFFFFF:
            return "none"
        return f"#{self.r:02X}{self.g:02X}{self.b:02X}"

    @classmethod
    def from_bytes(cls, data: bytes, offset: int = 0) -> "Color4Byte":
        """바이트에서 생성"""
        value = struct.unpack_from('<I', data, offset)[0]
        return cls(value)
```

---

## 10. 리스크 및 고려사항

### 10.1 라이선스

| 라이브러리 | 라이선스 | 상용 사용 |
|-----------|---------|----------|
| olefile | BSD-2-Clause | ✅ 가능 |
| lxml | BSD | ✅ 가능 |
| cryptography | Apache-2.0/BSD | ✅ 가능 |
| 본 프로젝트 | Apache-2.0 | ✅ 가능 |

### 10.2 기술적 위험

| 위험 | 설명 | 완화 방안 |
|------|------|----------|
| **HWP 스펙 불완전** | 공식 문서가 일부 불완전함 | hwplib 참조, 실제 파일 분석 |
| **암호화 파일** | 암호화된 HWP 미지원 | 선택적 의존성으로 분리 |
| **복잡한 객체** | 일부 그래픽 객체 복잡 | 단계별 구현 |
| **버전 호환성** | HWP 버전별 차이 존재 | 5.0 이상 집중 |

### 10.3 테스트 전략

1. **단위 테스트**: 각 파서/빌더 함수
2. **통합 테스트**: 전체 변환 파이프라인
3. **회귀 테스트**: 기존 Java 테스트 케이스 활용
4. **골든 테스트**: 참조 출력과 비교

```python
# tests/conftest.py
import pytest
from pathlib import Path

@pytest.fixture
def fixtures_dir() -> Path:
    return Path(__file__).parent / "fixtures"

@pytest.fixture
def sample_hwp(fixtures_dir: Path) -> Path:
    return fixtures_dir / "sample.hwp"
```

---

## 11. 참고 자료

### 공식 스펙 문서
- [한글문서파일형식 5.0 (revision 1.2)](https://cdn.hancom.com/link/docs/한글문서파일형식_5.0_revision1.2.pdf)
- [한글문서파일형식 5.0 (revision 1.3)](https://cdn.hancom.com/link/docs/한글문서파일형식_5.0_revision1.3.pdf)
- [한컴 지원센터 - OWPML 다운로드](https://www.hancom.com/support/downloadCenter/hwpOwpml)

### 오픈소스 라이브러리
- [hwplib (Java)](https://github.com/neolord0/hwplib) - HWP 읽기/쓰기
- [hwpxlib (Java)](https://github.com/neolord0/hwpxlib) - HWPX 읽기/쓰기
- [hwp2hwpx (Java)](https://github.com/neolord0/hwp2hwpx) - 변환기
- [pyhwp (Python)](https://github.com/mete0r/pyhwp) - HWP 파서 (AGPL)
- [olefile (Python)](https://github.com/decalage2/olefile) - OLE/CFB 파일 처리

### Python 도구
- [uv 공식 문서](https://docs.astral.sh/uv/)
- [Python struct 모듈](https://docs.python.org/3/library/struct.html)
- [lxml 문서](https://lxml.de/)
- [olefile 문서](https://olefile.readthedocs.io/)

### 기술 문서
- [한컴테크 - HWP 포맷 구조](https://tech.hancom.com/한-글-문서-파일-형식-hwp-포맷-구조-살펴보기/)
- [한컴테크 - HWPX 포맷 구조](https://tech.hancom.com/hwpxformat/)
- [NHN Cloud - HWPX 살펴보기](https://meetup.nhncloud.com/posts/311)

---

## 12. 결론 및 다음 단계

### 권장 기술 스택 (외부 의존 없이 직접 구현)

| 영역 | 선택 | 이유 |
|------|------|------|
| **패키지 매니저** | uv | 빠른 속도, 현대적 기능 |
| **Python 버전** | 3.10+ | 타입 힌팅 강화, 패턴 매칭 |
| **CFB 파일 처리** | olefile | 표준, BSD 라이선스 |
| **XML 처리** | lxml | 네임스페이스 지원, 성능 |
| **바이너리 파싱** | struct (표준) | 내장 모듈 |
| **압축** | zlib (표준) | 내장 모듈 |
| **타입 체크** | mypy | 정적 타입 분석 |
| **포매터/린터** | ruff | 빠른 속도, 통합 도구 |
| **테스트** | pytest | Python 표준 |

### 예상 작업량 (외부 라이브러리 직접 구현 포함)

| Phase | 내용 | 예상 코드량 |
|-------|------|------------|
| Phase 1 | 기초 인프라 | ~1,500줄 |
| Phase 2 | DocInfo 변환 | ~2,500줄 |
| Phase 3 | BodyText 변환 | ~3,000줄 |
| Phase 4 | 고급 기능 | ~2,000줄 |
| Phase 5 | 테스트/문서 | ~1,000줄 |
| **합계** | | **~10,000줄** |

### 체크리스트

- [x] 기술 리서치 완료
- [x] HWP 파일 포맷 스펙 조사
- [x] HWPX 파일 포맷 스펙 조사
- [x] hwplib 구조 분석
- [x] hwpxlib 구조 분석
- [x] Python 기반 라이브러리 선정
- [ ] uv 프로젝트 초기화
- [ ] HWP 레코드 파서 구현
- [ ] HWP 데이터 모델 구현
- [ ] HWPX 데이터 모델 구현
- [ ] HWPX Writer 구현
- [ ] 변환기 구현
- [ ] 테스트 작성
