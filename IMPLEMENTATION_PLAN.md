# HWP2HWPX Python 구현 계획서

## 1. 프로젝트 개요

### 1.1 목표
Java 기반 hwp2hwpx 프로젝트를 Python + uv 기반으로 완전히 재구현합니다.
외부 의존 라이브러리(hwplib, hwpxlib)도 Python으로 직접 구현합니다.

### 1.2 범위
- **hwplib 대체**: HWP 파일 읽기 (CFB/OLE2 파싱 + zlib 압축 해제)
- **hwpxlib 대체**: HWPX 파일 쓰기 (ZIP + XML 생성)
- **hwp2hwpx**: HWP → HWPX 변환 로직

### 1.3 기술 스택
| 영역 | 기술 |
|------|------|
| 언어 | Python 3.10+ |
| 패키지 매니저 | uv |
| CFB 파일 처리 | olefile |
| XML 처리 | lxml |
| 바이너리 파싱 | struct (표준) |
| 압축 | zlib (표준) |
| 타입 체크 | mypy |
| 린터 | ruff |
| 테스트 | pytest |

---

## 2. 구현 단계 (총 5 Phase)

### Phase 0: 프로젝트 초기화 (1일)

#### 작업 항목
- [ ] **P0-1**: uv 프로젝트 생성
  ```bash
  uv init py-hwp2hwpx --package --build-backend hatchling
  ```
- [ ] **P0-2**: pyproject.toml 설정
- [ ] **P0-3**: 디렉토리 구조 생성
- [ ] **P0-4**: 기본 의존성 추가 (olefile, lxml)
- [ ] **P0-5**: 개발 의존성 추가 (pytest, ruff, mypy)
- [ ] **P0-6**: .gitignore, README.md 작성

#### 산출물
```
src/hwp2hwpx/
├── __init__.py
├── py.typed
├── hwp/
│   └── __init__.py
├── hwpx/
│   └── __init__.py
├── transform/
│   └── __init__.py
└── util/
    └── __init__.py
tests/
└── __init__.py
```

---

### Phase 1: 기반 인프라 (3-4일)

#### 1.1 HWP 열거형 및 상수 정의

- [ ] **P1-1**: `hwp/enums/tags.py` - HWP 태그 ID 정의
  ```python
  class HWPTag(IntEnum):
      DOCUMENT_PROPERTIES = 0x10
      ID_MAPPINGS = 0x11
      BIN_DATA = 0x12
      FACE_NAME = 0x13
      # ... 약 30개 태그
  ```

- [ ] **P1-2**: `hwp/enums/char_shape.py` - 글자 모양 관련 열거형
  ```python
  class UnderlineType(Enum): ...
  class StrikeoutType(Enum): ...
  class OutlineType(Enum): ...
  class ShadowType(Enum): ...
  class EmphasisType(Enum): ...
  ```

- [ ] **P1-3**: `hwp/enums/para_shape.py` - 문단 모양 관련 열거형
  ```python
  class Alignment(Enum): ...
  class VerticalAlignment(Enum): ...
  class LineSpacingType(Enum): ...
  ```

- [ ] **P1-4**: `hwp/enums/border_fill.py` - 테두리/채우기 관련 열거형
  ```python
  class BorderType(Enum): ...
  class BorderThickness(Enum): ...
  class FillType(Enum): ...
  ```

- [ ] **P1-5**: `hwp/enums/control.py` - 컨트롤 관련 열거형
  ```python
  class ControlType(Enum): ...
  class GsoType(Enum): ...
  class TableBorderFillType(Enum): ...
  ```

#### 1.2 유틸리티 모듈

- [ ] **P1-6**: `util/binary.py` - 바이너리 파싱 유틸리티
  ```python
  def read_uint8(data: bytes, offset: int) -> tuple[int, int]: ...
  def read_uint16(data: bytes, offset: int) -> tuple[int, int]: ...
  def read_uint32(data: bytes, offset: int) -> tuple[int, int]: ...
  def read_int32(data: bytes, offset: int) -> tuple[int, int]: ...
  def read_wchar_string(data: bytes, offset: int, length: int) -> tuple[str, int]: ...
  def read_hwpunit(data: bytes, offset: int) -> tuple[int, int]: ...
  ```

- [ ] **P1-7**: `util/compression.py` - 압축/해제 유틸리티
  ```python
  def decompress_stream(data: bytes) -> bytes: ...
  def compress_stream(data: bytes) -> bytes: ...
  def is_compressed(properties: int) -> bool: ...
  ```

- [ ] **P1-8**: `util/color.py` - 색상 처리 유틸리티
  ```python
  @dataclass
  class Color4Byte:
      value: int
      def to_hex(self) -> str: ...
      def is_none(self) -> bool: ...
  ```

#### 1.3 HWP 레코드 파서

- [ ] **P1-9**: `hwp/parser/record.py` - 레코드 파싱
  ```python
  @dataclass
  class RecordHeader:
      tag_id: int
      level: int
      size: int

  def parse_record_header(data: bytes, offset: int) -> tuple[RecordHeader, int]: ...
  def parse_all_records(data: bytes) -> list[tuple[RecordHeader, bytes]]: ...
  ```

#### 1.4 HWPX 열거형 정의

- [ ] **P1-10**: `hwpx/enums/line_type.py` - 선 종류
- [ ] **P1-11**: `hwpx/enums/align.py` - 정렬 타입
- [ ] **P1-12**: `hwpx/enums/number_type.py` - 번호 타입

#### 1.5 값 변환기

- [ ] **P1-13**: `util/value_converter.py` - HWP ↔ HWPX 값 변환
  ```python
  def border_type_to_line_type(hwp: BorderType) -> LineType2: ...
  def border_thickness_to_line_width(hwp: BorderThickness) -> LineWidth: ...
  def alignment_to_horizontal_align(hwp: Alignment) -> HorizontalAlign2: ...
  def color_to_hex(hwp: Color4Byte) -> str: ...
  # ... 약 30개 변환 함수
  ```

#### 산출물
- 열거형 ~15개 파일
- 유틸리티 ~5개 파일
- 약 1,500줄

---

### Phase 2: HWP 파일 읽기 (4-5일)

#### 2.1 HWP 데이터 모델

- [ ] **P2-1**: `hwp/models/file_header.py` - 파일 헤더
  ```python
  @dataclass
  class FileHeader:
      signature: str
      version: FileVersion
      properties: int

      @property
      def is_compressed(self) -> bool: ...
      @property
      def is_encrypted(self) -> bool: ...
  ```

- [ ] **P2-2**: `hwp/models/doc_info.py` - 문서 정보
  ```python
  @dataclass
  class DocInfo:
      document_properties: DocumentProperties
      id_mappings: IDMappings
      face_names: list[FaceName]
      border_fills: list[BorderFill]
      char_shapes: list[CharShape]
      tab_defs: list[TabDef]
      numberings: list[Numbering]
      bullets: list[Bullet]
      para_shapes: list[ParaShape]
      styles: list[Style]
      memo_shapes: list[MemoShape]
  ```

- [ ] **P2-3**: `hwp/models/face_name.py` - 글꼴 정보
- [ ] **P2-4**: `hwp/models/char_shape.py` - 글자 모양
- [ ] **P2-5**: `hwp/models/para_shape.py` - 문단 모양
- [ ] **P2-6**: `hwp/models/border_fill.py` - 테두리/채우기
- [ ] **P2-7**: `hwp/models/style.py` - 스타일
- [ ] **P2-8**: `hwp/models/numbering.py` - 번호 매기기
- [ ] **P2-9**: `hwp/models/bullet.py` - 글머리표

#### 2.2 BodyText 데이터 모델

- [ ] **P2-10**: `hwp/models/body_text.py` - 본문 컨테이너
  ```python
  @dataclass
  class BodyText:
      sections: list[Section]

  @dataclass
  class Section:
      paragraphs: list[Paragraph]
  ```

- [ ] **P2-11**: `hwp/models/paragraph.py` - 문단
  ```python
  @dataclass
  class Paragraph:
      header: ParaHeader
      text: ParaText
      char_shapes: list[ParaCharShape]
      line_segs: list[LineSeg]
      controls: list[Control]
  ```

- [ ] **P2-12**: `hwp/models/control.py` - 컨트롤 기본 클래스
- [ ] **P2-13**: `hwp/models/table.py` - 표
- [ ] **P2-14**: `hwp/models/gso.py` - 그래픽 객체
- [ ] **P2-15**: `hwp/models/bin_data.py` - 바이너리 데이터

#### 2.3 HWP 파서 구현

- [ ] **P2-16**: `hwp/parser/doc_info.py` - DocInfo 파싱
  ```python
  def parse_doc_info(data: bytes) -> DocInfo: ...
  def parse_face_name(data: bytes) -> FaceName: ...
  def parse_char_shape(data: bytes) -> CharShape: ...
  # ... 각 태그별 파서
  ```

- [ ] **P2-17**: `hwp/parser/body_text.py` - BodyText 파싱
  ```python
  def parse_section(data: bytes) -> Section: ...
  def parse_paragraph(records: list) -> Paragraph: ...
  def parse_control(records: list) -> Control: ...
  ```

- [ ] **P2-18**: `hwp/parser/table.py` - 표 파싱
- [ ] **P2-19**: `hwp/parser/gso.py` - 그래픽 객체 파싱

#### 2.4 HWP 리더

- [ ] **P2-20**: `hwp/reader.py` - 메인 리더
  ```python
  class HWPReader:
      @staticmethod
      def read(filepath: str | Path) -> HWPFile:
          ole = olefile.OleFileIO(filepath)
          file_header = parse_file_header(ole)
          doc_info = parse_doc_info(ole, file_header)
          body_text = parse_body_text(ole, file_header)
          bin_data = parse_bin_data(ole)
          return HWPFile(file_header, doc_info, body_text, bin_data)
  ```

- [ ] **P2-21**: `hwp/models/file.py` - 최상위 HWPFile
  ```python
  @dataclass
  class HWPFile:
      file_header: FileHeader
      doc_info: DocInfo
      body_text: BodyText
      bin_data: BinData
      summary_info: Optional[SummaryInfo] = None
  ```

#### 산출물
- 모델 ~15개 파일
- 파서 ~10개 파일
- 약 2,500줄

---

### Phase 3: HWPX 파일 쓰기 (3-4일)

#### 3.1 HWPX 데이터 모델

- [ ] **P3-1**: `hwpx/models/file.py` - HWPXFile
  ```python
  @dataclass
  class HWPXFile:
      version: VersionXML
      settings: Optional[SettingsXML]
      container: ContainerXML
      manifest: ManifestXML
      content_hpf: ContentHPF
      header: HeaderXML
      sections: list[SectionXML]
      master_pages: list[MasterPageXML]
      bin_data: dict[str, bytes]
  ```

- [ ] **P3-2**: `hwpx/models/version.py` - VersionXML
- [ ] **P3-3**: `hwpx/models/settings.py` - SettingsXML
- [ ] **P3-4**: `hwpx/models/container.py` - ContainerXML
- [ ] **P3-5**: `hwpx/models/manifest.py` - ManifestXML
- [ ] **P3-6**: `hwpx/models/content_hpf.py` - ContentHPF
- [ ] **P3-7**: `hwpx/models/header.py` - HeaderXML
- [ ] **P3-8**: `hwpx/models/section.py` - SectionXML
- [ ] **P3-9**: `hwpx/models/paragraph.py` - Para, Run, T 등
- [ ] **P3-10**: `hwpx/models/table.py` - Table, Row, Cell
- [ ] **P3-11**: `hwpx/models/shape.py` - 그래픽 도형들

#### 3.2 XML 빌더

- [ ] **P3-12**: `hwpx/builder/base.py` - XML 빌더 기본 클래스
  ```python
  class XMLBuilder:
      NAMESPACES = {
          'hh': 'http://www.hancom.co.kr/hwpml/2011/head',
          'hs': 'http://www.hancom.co.kr/hwpml/2011/section',
          'hp': 'http://www.hancom.co.kr/hwpml/2011/paragraph',
          'hc': 'http://www.hancom.co.kr/hwpml/2011/core',
      }

      def create_element(self, tag: str, ns: str = None, **attrs): ...
      def to_bytes(self, root: Element) -> bytes: ...
  ```

- [ ] **P3-13**: `hwpx/builder/version.py` - version.xml 빌더
- [ ] **P3-14**: `hwpx/builder/header.py` - header.xml 빌더
- [ ] **P3-15**: `hwpx/builder/section.py` - section.xml 빌더
- [ ] **P3-16**: `hwpx/builder/content_hpf.py` - content.hpf 빌더

#### 3.3 HWPX 라이터

- [ ] **P3-17**: `hwpx/writer.py` - 메인 라이터
  ```python
  class HWPXWriter:
      @staticmethod
      def write(hwpx: HWPXFile, filepath: str | Path) -> None:
          with zipfile.ZipFile(filepath, 'w', ZIP_DEFLATED) as zf:
              zf.writestr('mimetype', MIMETYPE)
              zf.writestr('version.xml', build_version_xml(hwpx.version))
              zf.writestr('META-INF/container.xml', build_container_xml(...))
              zf.writestr('META-INF/manifest.xml', build_manifest_xml(...))
              zf.writestr('Contents/content.hpf', build_content_hpf(...))
              zf.writestr('Contents/header.xml', build_header_xml(...))
              for i, section in enumerate(hwpx.sections):
                  zf.writestr(f'Contents/section{i}.xml', build_section_xml(...))
              # BinData 파일들
              for name, data in hwpx.bin_data.items():
                  zf.writestr(f'BinData/{name}', data)
  ```

#### 산출물
- 모델 ~12개 파일
- 빌더 ~6개 파일
- 약 2,000줄

---

### Phase 4: 변환기 구현 (5-7일)

#### 4.1 변환 인프라

- [ ] **P4-1**: `transform/parameter.py` - 변환 컨텍스트
  ```python
  @dataclass
  class Parameter:
      hwp: HWPFile
      hwpx: HWPXFile
      master_page_map: dict[int, MasterPageInfo] = field(default_factory=dict)
      bin_data_map: dict[int, str] = field(default_factory=dict)
      field_stack: list[FieldBegin] = field(default_factory=list)
  ```

- [ ] **P4-2**: `transform/converter.py` - 기본 변환기 클래스
  ```python
  class Converter:
      def __init__(self, param: Parameter):
          self.param = param

      @property
      def hwp(self) -> HWPFile: ...
      @property
      def hwpx(self) -> HWPXFile: ...
  ```

#### 4.2 메타데이터 변환

- [ ] **P4-3**: `transform/content_hpf.py` - ContentHPF 변환
  ```python
  class ContentHPFConverter(Converter):
      def convert(self) -> None:
          self._metadata()      # 문서 메타데이터
          self._manifest()      # 파일 목록
          self._spine()         # 읽기 순서
          self._bin_data()      # 바이너리 데이터 등록
  ```

#### 4.3 헤더 변환

- [ ] **P4-4**: `transform/header/header_xml.py` - 헤더 메인 변환기
  ```python
  class HeaderXMLConverter(Converter):
      def convert(self) -> None:
          self._begin_num()           # 번호 시작값
          self._ref_list()            # 참조 목록
          self._compatible_document() # 호환성
          self._doc_option()          # 문서 옵션
  ```

- [ ] **P4-5**: `transform/header/fontfaces.py` - 글꼴 변환
- [ ] **P4-6**: `transform/header/char_properties.py` - 글자 모양 변환
- [ ] **P4-7**: `transform/header/para_properties.py` - 문단 모양 변환
- [ ] **P4-8**: `transform/header/border_fills.py` - 테두리/채우기 변환
- [ ] **P4-9**: `transform/header/numberings.py` - 번호 매기기 변환
- [ ] **P4-10**: `transform/header/bullets.py` - 글머리표 변환
- [ ] **P4-11**: `transform/header/styles.py` - 스타일 변환
- [ ] **P4-12**: `transform/header/tab_properties.py` - 탭 변환

#### 4.4 섹션 변환

- [ ] **P4-13**: `transform/section/section_xml.py` - 섹션 메인 변환기
  ```python
  class SectionXMLConverter(Converter):
      def convert(self) -> None:
          for hwp_section in self.hwp.body_text.sections:
              section_xml = self.hwpx.add_section()
              for hwp_para in hwp_section.paragraphs:
                  ParaConverter(self.param).convert(section_xml, hwp_para)
  ```

- [ ] **P4-14**: `transform/section/para.py` - 문단 변환
  ```python
  class ParaConverter(Converter):
      def convert(self, section: SectionXML, hwp_para: Paragraph) -> None:
          para = section.add_para()
          para.para_pr_id_ref = hwp_para.header.para_shape_id
          para.style_id_ref = hwp_para.header.style_id
          self._runs(para, hwp_para)
          CharsConverter(self.param).convert(para, hwp_para)
          self._line_segs(para, hwp_para)
  ```

- [ ] **P4-15**: `transform/section/chars.py` - 문자/객체 변환 (가장 복잡)
  ```python
  class CharsConverter(Converter):
      def convert(self, para: Para, hwp_para: Paragraph) -> None:
          for char in hwp_para.text.chars:
              if char.type == CharType.NORMAL:
                  self._normal_char(char)
              elif char.type == CharType.CONTROL_CHAR:
                  self._control_char(char)
              elif char.type == CharType.CONTROL_INLINE:
                  self._inline_control(char)
              elif char.type == CharType.CONTROL_EXTEND:
                  self._extend_control(char, control_index)
  ```

- [ ] **P4-16**: `transform/section/sublist.py` - 서브리스트 변환
  ```python
  class SubListConverter(Converter):
      def convert_for_cell(self, cell: Cell, hwp_cell): ...
      def convert_for_header(self, header: Header, hwp_header): ...
      def convert_for_footer(self, footer: Footer, hwp_footer): ...
      def convert_for_footnote(self, footnote: Footnote, hwp_fn): ...
      def convert_for_caption(self, caption: Caption, hwp_cap): ...
  ```

#### 4.5 객체 변환

- [ ] **P4-17**: `transform/object/table.py` - 표 변환
  ```python
  class TableConverter(Converter):
      def convert(self, run: Run, hwp_table: ControlTable) -> None:
          table = run.add_table()
          self._shape_object(table, hwp_table)
          self._table_properties(table, hwp_table)
          self._rows(table, hwp_table)
  ```

- [ ] **P4-18**: `transform/object/cell.py` - 셀 변환
- [ ] **P4-19**: `transform/object/gso.py` - GSO 라우터
  ```python
  class GsoConverter(Converter):
      def convert(self, run: Run, hwp_gso: GsoControl) -> None:
          gso_type = hwp_gso.gso_type
          if gso_type == GsoType.LINE:
              LineConverter(self.param).convert(run, hwp_gso)
          elif gso_type == GsoType.RECTANGLE:
              RectangleConverter(self.param).convert(run, hwp_gso)
          # ... 나머지 타입들
  ```

- [ ] **P4-20**: `transform/object/line.py` - 선 변환
- [ ] **P4-21**: `transform/object/rectangle.py` - 직사각형 변환
- [ ] **P4-22**: `transform/object/ellipse.py` - 타원 변환
- [ ] **P4-23**: `transform/object/arc.py` - 호 변환
- [ ] **P4-24**: `transform/object/polygon.py` - 다각형 변환
- [ ] **P4-25**: `transform/object/curve.py` - 곡선 변환
- [ ] **P4-26**: `transform/object/picture.py` - 그림 변환
- [ ] **P4-27**: `transform/object/ole.py` - OLE 변환
- [ ] **P4-28**: `transform/object/container.py` - 컨테이너(그룹) 변환
- [ ] **P4-29**: `transform/object/textart.py` - 텍스트아트 변환
- [ ] **P4-30**: `transform/object/connect_line.py` - 연결선 변환

#### 4.6 기타 객체 변환

- [ ] **P4-31**: `transform/object/equation.py` - 수식 변환
- [ ] **P4-32**: `transform/object/header_footer.py` - 머리말/꼬리말 변환
- [ ] **P4-33**: `transform/object/footnote.py` - 각주/미주 변환
- [ ] **P4-34**: `transform/object/bookmark.py` - 책갈피 변환
- [ ] **P4-35**: `transform/object/field.py` - 필드 변환
- [ ] **P4-36**: `transform/object/form.py` - 폼 컨트롤 변환

#### 4.7 메인 변환기

- [ ] **P4-37**: `transform/hwp2hwpx.py` - 메인 변환 진입점
  ```python
  class Hwp2HwpxConverter:
      def __init__(self, hwp: HWPFile):
          self.hwp = hwp
          self.hwpx = HWPXFile()
          self.param = Parameter(hwp, self.hwpx)

      def convert(self) -> HWPXFile:
          self._version_xml()
          self._manifest_xml()
          self._container_xml()
          ContentHPFConverter(self.param).convert()
          HeaderXMLConverter(self.param).convert()
          MasterPageConverter(self.param).convert()
          SectionXMLConverter(self.param).convert()
          self._settings_xml()
          return self.hwpx

      @staticmethod
      def to_hwpx(hwp: HWPFile) -> HWPXFile:
          return Hwp2HwpxConverter(hwp).convert()
  ```

#### 산출물
- 변환기 ~40개 파일
- 약 3,500줄

---

### Phase 5: CLI 및 테스트 (2-3일)

#### 5.1 CLI 인터페이스

- [ ] **P5-1**: `cli.py` - 명령줄 인터페이스
  ```python
  import argparse
  from pathlib import Path
  from .hwp.reader import HWPReader
  from .transform.hwp2hwpx import Hwp2HwpxConverter
  from .hwpx.writer import HWPXWriter

  def main():
      parser = argparse.ArgumentParser(description='HWP to HWPX converter')
      parser.add_argument('input', type=Path, help='Input HWP file')
      parser.add_argument('output', type=Path, help='Output HWPX file')
      parser.add_argument('-v', '--verbose', action='store_true')
      args = parser.parse_args()

      hwp = HWPReader.read(args.input)
      hwpx = Hwp2HwpxConverter.to_hwpx(hwp)
      HWPXWriter.write(hwpx, args.output)
      print(f'Converted: {args.input} -> {args.output}')

  if __name__ == '__main__':
      main()
  ```

#### 5.2 테스트

- [ ] **P5-2**: `tests/conftest.py` - pytest 설정
  ```python
  import pytest
  from pathlib import Path

  @pytest.fixture
  def fixtures_dir() -> Path:
      return Path(__file__).parent / 'fixtures'

  @pytest.fixture
  def sample_hwp(fixtures_dir: Path) -> Path:
      return fixtures_dir / 'sample.hwp'
  ```

- [ ] **P5-3**: `tests/hwp/test_reader.py` - HWP 리더 테스트
- [ ] **P5-4**: `tests/hwp/test_parser.py` - 레코드 파서 테스트
- [ ] **P5-5**: `tests/hwpx/test_writer.py` - HWPX 라이터 테스트
- [ ] **P5-6**: `tests/hwpx/test_builder.py` - XML 빌더 테스트
- [ ] **P5-7**: `tests/transform/test_converter.py` - 변환기 테스트
- [ ] **P5-8**: `tests/test_integration.py` - 통합 테스트 (전체 변환)

#### 5.3 테스트 데이터 마이그레이션

- [ ] **P5-9**: 기존 Java 테스트 HWP 파일 복사
  - `test/` 디렉토리의 HWP 파일들을 `tests/fixtures/`로

- [ ] **P5-10**: 골든 테스트 설정
  - Java 버전으로 변환한 HWPX 파일을 참조 파일로 사용

#### 산출물
- CLI 1개 파일
- 테스트 ~10개 파일
- 약 1,000줄

---

## 3. 의존성 그래프

```
Phase 0: 프로젝트 초기화
    ↓
Phase 1: 기반 인프라
    ├── 열거형 정의
    ├── 유틸리티
    └── 레코드 파서
    ↓
Phase 2: HWP 파일 읽기
    ├── 데이터 모델 (Phase 1 열거형 의존)
    ├── 파서 (Phase 1 유틸리티, 레코드 파서 의존)
    └── 리더 (모델, 파서 의존)
    ↓
Phase 3: HWPX 파일 쓰기
    ├── 데이터 모델 (Phase 1 열거형 의존)
    ├── XML 빌더 (모델 의존)
    └── 라이터 (모델, 빌더 의존)
    ↓
Phase 4: 변환기 구현
    ├── Parameter (Phase 2, 3 모델 의존)
    ├── 헤더 변환 (Phase 1-3 모두 의존)
    ├── 섹션 변환 (헤더 변환 의존)
    └── 객체 변환 (섹션 변환 의존)
    ↓
Phase 5: CLI 및 테스트
    └── 전체 통합
```

---

## 4. 파일별 의존성

### 핵심 의존성 체인

```
cli.py
  └── transform/hwp2hwpx.py
        ├── hwp/reader.py
        │     ├── hwp/parser/*.py
        │     │     └── hwp/models/*.py
        │     │           └── hwp/enums/*.py
        │     └── util/binary.py, compression.py
        │
        ├── hwpx/writer.py
        │     ├── hwpx/builder/*.py
        │     │     └── hwpx/models/*.py
        │     │           └── hwpx/enums/*.py
        │     └── util/color.py
        │
        └── transform/*.py
              └── util/value_converter.py
```

---

## 5. 구현 순서 권장

### 수직 슬라이스 접근법

가장 단순한 케이스부터 전체 파이프라인을 구현:

#### Slice 1: 빈 문서 (1일)
1. FileHeader 파싱
2. 빈 HWPXFile 생성
3. version.xml, container.xml 출력
4. 테스트: 빈 HWP → 빈 HWPX

#### Slice 2: 텍스트만 있는 문서 (2일)
1. DocInfo 파싱 (글꼴, 글자/문단 모양)
2. BodyText 파싱 (일반 텍스트만)
3. header.xml 생성
4. section.xml 생성 (텍스트만)
5. 테스트: 텍스트만 있는 HWP → HWPX

#### Slice 3: 표가 있는 문서 (2일)
1. Table 파싱
2. Table 변환
3. 테스트: 표가 있는 HWP → HWPX

#### Slice 4: 그림이 있는 문서 (2일)
1. Picture 파싱
2. BinData 처리
3. Picture 변환
4. 테스트: 그림이 있는 HWP → HWPX

#### Slice 5: 나머지 객체들 (5-7일)
1. 각 객체 타입별 구현
2. 테스트 추가

---

## 6. 예상 일정

| Phase | 작업 | 예상 기간 | 누적 |
|-------|------|----------|------|
| Phase 0 | 프로젝트 초기화 | 1일 | 1일 |
| Phase 1 | 기반 인프라 | 3-4일 | 5일 |
| Phase 2 | HWP 파일 읽기 | 4-5일 | 10일 |
| Phase 3 | HWPX 파일 쓰기 | 3-4일 | 14일 |
| Phase 4 | 변환기 구현 | 5-7일 | 21일 |
| Phase 5 | CLI 및 테스트 | 2-3일 | 24일 |
| **합계** | | **18-24일** | |

---

## 7. 예상 코드량

| 영역 | 파일 수 | 예상 줄 |
|------|--------|--------|
| hwp/enums | ~8 | ~500 |
| hwp/models | ~15 | ~1,500 |
| hwp/parser | ~10 | ~1,500 |
| hwp/reader | ~2 | ~200 |
| hwpx/enums | ~5 | ~300 |
| hwpx/models | ~12 | ~1,000 |
| hwpx/builder | ~6 | ~800 |
| hwpx/writer | ~2 | ~200 |
| transform | ~40 | ~3,500 |
| util | ~5 | ~400 |
| cli | ~1 | ~100 |
| tests | ~10 | ~1,000 |
| **합계** | **~115** | **~11,000** |

---

## 8. 리스크 관리

### 높은 리스크
| 리스크 | 영향 | 완화 방안 |
|--------|------|----------|
| HWP 스펙 불완전 | 파싱 오류 | hwplib 소스 참조 |
| 복잡한 객체 | 구현 지연 | 단계별 구현 |
| 레코드 크기 불일치 | 파싱 실패 | 유연한 파서 구현 |

### 중간 리스크
| 리스크 | 영향 | 완화 방안 |
|--------|------|----------|
| XML 네임스페이스 | 잘못된 출력 | 실제 HWPX 파일 분석 |
| 값 변환 오류 | 잘못된 스타일 | 단위 테스트 강화 |

### 낮은 리스크
| 리스크 | 영향 | 완화 방안 |
|--------|------|----------|
| 성능 이슈 | 느린 변환 | 프로파일링 후 최적화 |

---

## 9. 체크리스트

### 시작 전
- [ ] Python 3.10+ 설치 확인
- [ ] uv 설치 확인
- [ ] 테스트용 HWP 파일 준비

### Phase 완료 기준
- [ ] Phase 0: 프로젝트 빌드 성공
- [ ] Phase 1: 모든 열거형/유틸리티 정의 완료
- [ ] Phase 2: 샘플 HWP 파일 파싱 성공
- [ ] Phase 3: 빈 HWPX 파일 생성 성공
- [ ] Phase 4: 전체 변환 파이프라인 동작
- [ ] Phase 5: 모든 테스트 통과

### 완료 후
- [ ] 문서화 완료
- [ ] PyPI 배포 준비
- [ ] CI/CD 설정

---

## 10. 참고 파일

- `RESEARCH_JAVA_TO_PYTHON_MIGRATION.md`: 기술 리서치
- Java 소스: `src/main/java/kr/dogfoot/hwp2hwpx/`
- 테스트 데이터: `test/*/from.hwp`
