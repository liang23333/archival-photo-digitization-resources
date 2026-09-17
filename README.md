# Archival Photo Digitization Resources & Best Practices

A comprehensive guide and curated directory of open-source tools, technical standards, and image processing best practices for museums, archives, libraries, and cultural heritage institutions digitizing photographic collections (prints, negatives, lantern slides, and glass plate negatives).

---

## Table of Contents

1. [Archival Digitization Fundamentals](#archival-digitization-fundamentals)
   - [FADGI & International Standards](#fadgi--international-standards)
   - [Preservation Masters vs. Access Derivatives](#preservation-masters-vs-access-derivatives)
2. [Pillow (Python) Best Practices for Archival Imaging](#pillow-python-best-practices-for-archival-imaging)
   - [1. File Formats & Compression (TIFF / BigTIFF)](#1-file-formats--compression-tiff--bigtiff)
   - [2. Color Management & ICC Profiles](#2-color-management--icc-profiles)
   - [3. Spatial Resolution & DPI Handling](#3-spatial-resolution--dpi-handling)
   - [4. Resampling & Derivative Generation](#4-resampling--derivative-generation)
   - [5. Metadata Preservation (EXIF, IPTC, TIFF Tags)](#5-metadata-preservation-exif-iptc-tiff-tags)
   - [6. High Bit-Depth (16-bit per channel) Integrity](#6-high-bit-depth-16-bit-per-channel-integrity)
   - [7. Metadata Sanitization for Public Web Derivatives](#7-metadata-sanitization-for-public-web-derivatives)
3. [Production Python Workflow Example](#production-python-workflow-example)
4. [Curated Open-Source Cultural Heritage Tool Ecosystem](#curated-open-source-cultural-heritage-tool-ecosystem)
   - [IIIF Viewers & Comparison Tools](#iiif-viewers--comparison-tools)
   - [IIIF Image Servers & Tile Delivery](#iiif-image-servers--tile-delivery)
   - [Digital Asset Management (DAM) & Repositories](#digital-asset-management-dam--repositories)
   - [Image Processing Engines & RAW Processing](#image-processing-engines--raw-processing)
   - [Metadata, Quality Assurance & Format Validation](#metadata-quality-assurance--format-validation)
5. [Contributing & Community Standards](#contributing--community-standards)

---

## Archival Digitization Fundamentals

### FADGI & International Standards
Cultural heritage digitization should adhere to recognized institutional guidelines:
- **FADGI (Federal Agencies Digital Guidelines Initiative)**: 
  - **4-Star Tier**: Maximum quality, strict spatial frequency response (SFR), color accuracy ($\Delta E < 2$), tonal reproduction, and no sharpening/lossy compression.
  - **3-Star Tier**: Recommended baseline for large-scale institutional photographic digitization.
- **Metamorfoze Preservation Imaging Guidelines**: International standard widely used in European heritage institutions.
- **ISO 19264-1**: Photography — Digital image capture, archival imaging systems and quality analysis.

### Preservation Masters vs. Access Derivatives

| Attribute | Preservation Master (Archival Copy) | Access / Service Derivative |
| :--- | :--- | :--- |
| **File Format** | Uncompressed TIFF or LZW-compressed TIFF (`.tif`, `.tiff`), BigTIFF | JPEG, WebP, AVIF, Pyramidal TIFF (`ptif`), JP2 |
| **Color Space** | Adobe RGB (1998) or ProPhoto RGB (ROMM) with embedded ICC | sRGB (for browser & display uniformity) |
| **Bit Depth** | 16-bit per channel (or 8-bit uncompressed minimum) | 8-bit per channel |
| **Compression** | Lossless (None, LZW, or Deflate) | Lossy or Lossless (quality 80–90) |
| **Resolution** | Native optical resolution (typically 300 to 1200+ DPI depending on media) | Downsampled for web/display (72–150 DPI) |
| **Edits / Crops** | Raw capture including target borders; no destructive editing | Cropped, contrast-adjusted, watermarked (if applicable) |

---

## Pillow (Python) Best Practices for Archival Imaging

[Pillow (PIL fork)](https://python-pillow.org/) is an industry standard for Python image processing scripts and automation pipelines. Below are key best practices when using Pillow on archival collections.

### 1. File Formats & Compression (TIFF / BigTIFF)
- **Lossless Only for Masters**: Never use lossy compression (such as JPEG-compressed TIFF) for preservation master files.
- **LZW or Deflate**: Pillow supports `compression="tiff_lzw"` or `compression="tiff_deflate"`. LZW is non-destructive and widely compatible across legacy institutional tools.
- **BigTIFF Support**: High-resolution film scans (e.g. 8x10 large format at 2400 DPI) easily exceed the standard 4 GB TIFF boundary. In Pillow, pass `big_tiff=True` to write standard 64-bit BigTIFF headers:
  ```python
  from PIL import Image

  # Saving an archival master as BigTIFF with lossless LZW compression
  master_image.save(
      "preservation_master.tif",
      format="TIFF",
      compression="tiff_lzw",
      big_tiff=True
  )
  ```

### 2. Color Management & ICC Profiles
Archival scans must retain their exact color profiles. Stripping an ICC profile will result in color shifts, incorrect contrast, and loss of tonal accuracy.

- **Preserve Embedded ICC Profiles**:
  Pillow stores the raw ICC profile in `im.info.get("icc_profile")`. When writing derivatives or transferring masters, pass this back explicitly:
  ```python
  icc_profile = im.info.get("icc_profile")

  # Retain existing profile when writing an output image
  im.save("output.tif", icc_profile=icc_profile)
  ```
- **Proper Color Space Conversion for Access Files**:
  When converting an archival master (Adobe RGB / ProPhoto RGB) into an sRGB derivative for web viewers, do not simply use `im.convert("RGB")` as this does not recalculate colorimetric values. Use `PIL.ImageCms` (LittleCMS backend) to execute perceptual or relative colorimetric transformations:
  ```python
  from PIL import Image, ImageCms
  import io

  def convert_to_srgb(im: Image.Image) -> Image.Image:
      icc = im.info.get("icc_profile")
      if not icc:
          # Already assumed sRGB or untagged
          return im

      src_profile = ImageCms.getOpenProfile(io.BytesIO(icc))
      srgb_profile = ImageCms.createProfile("sRGB")
      
      # Transform in-place or return converted image
      transformed = ImageCms.profileToProfile(
          im,
          src_profile,
          srgb_profile,
          renderingIntent=ImageCms.Intent.RELATIVE_COLORIMETRIC,
          outputMode="RGB"
      )
      return transformed
  ```

### 3. Spatial Resolution & DPI Handling
Archival images require accurate physical dimensions to enable 1:1 scale reproduction and cataloging measurements.
- Always preserve and set the `dpi` tuple and `resolution_unit`:
  ```python
  # Ensure DPI and measurement units (2 = inches, 3 = centimeters) are saved
  dpi_val = im.info.get("dpi", (600.0, 600.0))
  im.save(
      "derivative.jpg",
      format="JPEG",
      dpi=dpi_val,
      quality=90
  )
  ```

### 4. Resampling & Derivative Generation
When downsampling high-resolution photographic masters for web or IIIF thumbnails:
- **Avoid `NEAREST` or `BOX`**: These introduce severe aliasing, moiré patterns in film grain, and jagged edges.
- **Use `Resampling.LANCZOS`**: High-quality sinc interpolation that preserves photographic sharpness and fine lines without ringing artifacts:
  ```python
  from PIL import Image

  # Generate high-quality web thumbnail
  thumbnail = im.resize((1200, 800), resample=Image.Resampling.LANCZOS)
  ```

### 5. Metadata Preservation (EXIF, IPTC, TIFF Tags)
Preservation masters contain critical capture metadata (camera/scanner hardware, exposure, lens, scan date, operator notes).
- **Reading EXIF and TIFF Tags**:
  ```python
  from PIL import Image
  from PIL.ExifTags import TAGS

  with Image.open("scan.tif") as im:
      # Access EXIF dictionary
      exif_data = im.getexif()
      for tag_id, value in exif_data.items():
          tag_name = TAGS.get(tag_id, tag_id)
          print(f"{tag_name}: {value}")
  ```
- **Writing Custom Archival Tags**:
  Pillow allows saving standard and custom TIFF tags (e.g. `Artist`, `Copyright`, `ImageDescription`):
  ```python
  from PIL.TiffTags import TAGS

  custom_tags = {
      270: "Glass plate negative: Central Station 1912",  # ImageDescription
      315: "Digitization Lab Technician",                # Artist
      33432: "Public Domain / CC0",                      # Copyright
  }

  im.save("master_tagged.tif", tiffinfo=custom_tags, compression="tiff_lzw")
  ```

### 6. High Bit-Depth (16-bit per channel) Integrity
- 16-bit per channel images (48-bit RGB or 16-bit grayscale `mode="I;16"`) contain 65,536 tonal levels per channel vs. 256 levels in 8-bit.
- Never downconvert to 8-bit during intermediate processing stages, as this permanently compresses shadow and highlight dynamic range.
- Pillow can read and manipulate 16-bit grayscale (`I;16`) and 16-bit multi-channel files. When complex multi-channel 16-bit batch manipulation is required, pair Pillow with **pyvips** or **rawpy**.

### 7. Metadata Sanitization for Public Web Derivatives
While preservation masters should retain all hardware serial numbers, technician info, and internal tags, public-facing web deliverables may require GPS or internal workflow metadata scrubbing:
```python
# Explicitly strip sensitive metadata when writing public access copies
im.save(
    "public_access.jpg",
    format="JPEG",
    quality=85,
    optimize=True,
    exif=b"",
    icc_profile=None
)
```

---

## Production Python Workflow Example

Below is a production-ready script snippet implementing the above best practices to ingest a high-resolution archival master and produce a standardized preservation copy and an access derivative.

```python
#!/usr/bin/env python3
"""
archival_processor.py
Batch processing pipeline for archival photograph digitization.
"""
from pathlib import Path
import io
from PIL import Image, ImageCms, ImageOps

def process_archival_photo(
    input_path: Path,
    preservation_dir: Path,
    access_dir: Path
):
    preservation_dir.mkdir(parents=True, exist_ok=True)
    access_dir.mkdir(parents=True, exist_ok=True)

    with Image.open(input_path) as im:
        # Correct orientation based on EXIF tag if present without stripping other metadata
        im = ImageOps.exif_transpose(im)
        
        # 1. Capture original metadata & ICC profile
        icc_profile = im.info.get("icc_profile")
        dpi_val = im.info.get("dpi", (300.0, 300.0))
        exif = im.getexif()

        # 2. Save Standardized Preservation Master (TIFF, LZW, BigTIFF enabled)
        master_dest = preservation_dir / f"{input_path.stem}_master.tif"
        im.save(
            master_dest,
            format="TIFF",
            compression="tiff_lzw",
            big_tiff=True,
            dpi=dpi_val,
            icc_profile=icc_profile,
            exif=exif
        )
        print(f"[Preserved] {master_dest.name}")

        # 3. Generate Service / Web Access Derivative (sRGB, Resampled, JPEG)
        access_im = im.copy()
        
        # Convert color profile to sRGB for web compatibility
        if icc_profile:
            try:
                src_prof = ImageCms.getOpenProfile(io.BytesIO(icc_profile))
                srgb_prof = ImageCms.createProfile("sRGB")
                access_im = ImageCms.profileToProfile(
                    access_im,
                    src_prof,
                    srgb_prof,
                    renderingIntent=ImageCms.Intent.PERCEPTUAL,
                    outputMode="RGB"
                )
            except Exception as e:
                print(f"ICC conversion fallback: {e}")
                if access_im.mode != "RGB":
                    access_im = access_im.convert("RGB")
        elif access_im.mode != "RGB":
            access_im = access_im.convert("RGB")

        # Downsample for web access (max dimension 2048px)
        max_dim = 2048
        if max(access_im.size) > max_dim:
            scale = max_dim / max(access_im.size)
            new_size = (int(access_im.width * scale), int(access_im.height * scale))
            access_im = access_im.resize(new_size, resample=Image.Resampling.LANCZOS)

        access_dest = access_dir / f"{input_path.stem}_access.jpg"
        access_im.save(
            access_dest,
            format="JPEG",
            quality=85,
            optimize=True,
            dpi=(72.0, 72.0)
        )
        print(f"[Derivative] {access_dest.name}")

if __name__ == "__main__":
    sample_file = Path("scans/glass_plate_001.tif")
    if sample_file.exists():
        process_archival_photo(
            sample_file,
            preservation_dir=Path("archive/masters"),
            access_dir=Path("archive/access")
        )
```

---

## Curated Open-Source Cultural Heritage Tool Ecosystem

### IIIF Viewers & Comparison Tools
[International Image Interoperability Framework (IIIF)](https://iiif.io/) enables uniform access and deep-zoom comparison across distributed cultural repositories.

- **[ProjectMirador/mirador](https://github.com/ProjectMirador/mirador)**: Multi-window, deep-zoom web image viewer. Widely adopted by museums (e.g., Harvard, Stanford, BnF) for comparing side-by-side photographic prints, negatives, and multispectral captures, with full annotation and IIIF Presentation 3.0 support.
- **[openseadragon/openseadragon](https://github.com/openseadragon/openseadragon)**: Pure JavaScript high-resolution, zoomable image viewer for web applications. The foundational engine powering numerous custom museum viewers and tile visualizers.
- **[UniversalViewer/universalviewer](https://github.com/UniversalViewer/universalviewer)**: Multi-format viewer supporting IIIF image manifests, 3D models, audio, video, and PDF documents. Used by the British Library and National Library of Wales.
- **[tify-iiif-viewer/tify](https://github.com/tify-iiif-viewer/tify)**: Fast, modern, responsive, and mobile-friendly IIIF 2.x and 3.0 document and photo viewer built with Vue.js.
- **[DDMAL/diva.js](https://github.com/DDMAL/diva.js)**: High-performance IIIF image and document viewer designed for continuous scrolling across hundreds of high-resolution images.

---

### IIIF Image Servers & Tile Delivery
Image servers dynamically generate deep-zoom image tiles on demand from master pyramidal TIFFs or JPEG 2000 files without pre-generating millions of static tile files.

- **[cantaloupe-project/cantaloupe](https://github.com/cantaloupe-project/cantaloupe)**: The primary dynamic IIIF image server for cultural heritage institutions. Written in Java; supports Pyramidal TIFF, JPEG 2000, WebP, caching, watermarks, dynamic color space conversion, and ICC profile embedding.
- **[ruven/iipsrv](https://github.com/ruven/iipsrv)**: Ultra-fast streaming C++ image server supporting IIIF Image API, Zoomify, and DeepZoom protocols, optimized for gigapixel archival photographs and maps.
- **[loris-imageserver/loris](https://github.com/loris-imageserver/loris)**: Python-based IIIF Image API 2.0/2.1 server designed specifically for libraries and archives.
- **[samvera/serverless-iiif](https://github.com/samvera/serverless-iiif)**: Serverless IIIF Image API 2.1 and 3.0 implementation running on AWS Lambda with libvips, eliminating the need to manage dedicated server infrastructure.

---

### Digital Asset Management (DAM) & Repositories
Systems designed to catalog, organize, ingest, and preserve digital collections according to archival standards (e.g. OAIS, Dublin Core, MODS).

- **[omeka/omeka-s](https://github.com/omeka/omeka-s)**: Web publication platform designed specifically for galleries, libraries, archives, and museums (GLAM). Emphasizes linked open data, customizable metadata schemas, and native IIIF integration.
- **[artefactual/archivematica](https://github.com/artefactual/archivematica)**: OAIS-compliant open-source digital preservation system. Automatically packages files into Archival Information Packages (AIPs) and Dissemination Information Packages (DIPs) with checksum verification, format migration, and METS metadata.
- **[samvera/hyrax](https://github.com/samvera/hyrax)**: Ruby on Rails-based repository platform powering institutional repositories at major research libraries and universities.
- **[DSpace/DSpace](https://github.com/DSpace/DSpace)**: Industry-standard digital repository system for managing academic and cultural digital assets with long-term preservation workflows and OAI-PMH harvest support.
- **[resourcespace/resourcespace](https://github.com/resourcespace/resourcespace)**: Open-source DAM designed for heritage institutions, NGOs, and museums to catalog photographs, audio/video, and artwork.

---

### Image Processing Engines & RAW Processing
High-throughput tools for batch converting archival collections and generating derivatives.

- **[libvips/libvips](https://github.com/libvips/libvips)** & **[libvips/pyvips](https://github.com/libvips/pyvips)**: Extremely fast, memory-efficient image processing library. Unlike Pillow, libvips streams images in tiles, making it ideal for processing multi-gigabyte scans and building DeepZoom/Pyramidal TIFF pyramids (`vips dzsave`).
- **[letmaik/rawpy](https://github.com/letmaik/rawpy)**: Python wrapper for LibRaw. Indispensable for digitization studios capturing photographic negatives or prints directly using RAW camera sensors (e.g. Sony, Canon, Nikon, Phase One).
- **[python-pillow/pillow](https://github.com/python-pillow/pillow)**: The de-facto Python imaging library for automated format identification, EXIF extraction, thumbnail generation, and quality assurance scripts.

---

### Metadata, Quality Assurance & Format Validation
Ensuring archival files are valid according to ISO standards and that embedded metadata complies with preservation protocols.

- **[exiftool/exiftool](https://github.com/exiftool/exiftool)**: Phil Harvey's standard command-line and library for reading, writing, and manipulating EXIF, IPTC, XMP, MakerNotes, and ICC profile metadata.
- **[openpreserve/jhove](https://github.com/openpreserve/jhove)**: Format identification, validation, and characterization tool maintained by the Open Preservation Foundation. Validates TIFF, JPEG 2000, PDF, and WAV against formal specifications to ensure file longevity.
- **[openpreserve/jpylyzer](https://github.com/openpreserve/jpylyzer)**: Specialized JPEG 2000 (JP2 Part 1) validator and feature extractor used by national libraries and archives.
- **[openpreserve/fido](https://github.com/openpreserve/fido)**: Format Identification for Digital Objects (FIDO), command-line format identification matching against PRONOM technical registries.
- **[LibraryOfCongress/embARC](https://github.com/LibraryOfCongress/embARC)**: Created by FADGI (Federal Agencies Digital Guidelines Initiative) for managing and embedding archival metadata in TIFF, DPX, and BWF files.

---

## Contributing & Community Standards

We welcome contributions, updates on institutional workflows, and additional tool recommendations! Please submit an issue or pull request following standard GitHub conventions.

### References & Further Reading
- [FADGI Technical Guidelines for Digitizing Cultural Heritage Materials](https://www.digitizationguidelines.gov/guidelines/digitize-technical.html)
- [International Image Interoperability Framework (IIIF) Specifications](https://iiif.io/api/)
- [Open Preservation Foundation Knowledge Base](https://openpreservation.org/)
- [Pillow Documentation: Image File Formats & Concepts](https://pillow.readthedocs.io/)
