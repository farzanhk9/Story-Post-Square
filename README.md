#!/usr/bin/env python3
# File: ig_batch_tool.py
# Batch tool for Instagram assets: resize (story/post/square), optimize, watermark, strip EXIF, rename.

import argparse, os, sys, glob
from typing import Tuple, Optional
from PIL import Image, ImageOps
import piexif

PRESETS = {
    "story": (1080, 1920),
    "post": (1080, 1350),
    "square": (1080, 1080),
}

def ensure_dir(p: str):
    os.makedirs(p, exist_ok=True)

def load_image(path: str) -> Image.Image:
    img = Image.open(path)
    if img.mode not in ("RGB", "RGBA"):
        img = img.convert("RGBA" if "A" in img.getbands() else "RGB")
    return img

def fit_canvas(img: Image.Image, target: Tuple[int,int], safe_pad: int=0, bgcolor=(0,0,0)) -> Image.Image:
    W,H = target
    # Letterbox with background to exactly match target
    bg = Image.new("RGB", (W,H), bgcolor)
    # contain
    ratio = min((W - 2*safe_pad)/img.width, (H - 2*safe_pad)/img.height)
    new_w, new_h = max(1, int(img.width*ratio)), max(1, int(img.height*ratio))
    resized = img.resize((new_w, new_h), Image.LANCZOS)
    x = (W - new_w)//2
    y = (H - new_h)//2
    if resized.mode == "RGBA":
        bg = bg.convert("RGBA")
    bg.paste(resized, (x,y), resized if resized.mode == "RGBA" else None)
    return bg

def resize_longest(img: Image.Image, longest: int) -> Image.Image:
    w,h = img.size
    if max(w,h) <= longest:
        return img
    r = longest / max(w,h)
    return img.resize((int(w*r), int(h*r)), Image.LANCZOS)

def place_logo(base: Image.Image, logo_path: str, pos: str, scale: float, margin: int) -> Image.Image:
    if not logo_path or not os.path.exists(logo_path):
        return base
    logo = load_image(logo_path).convert("RGBA")
    max_w = int(base.width * scale)
    r = max_w / logo.width
    logo = logo.resize((max(1,int(logo.width*r)), max(1,int(logo.height*r))), Image.LANCZOS)
    x, y = margin, margin
    if "right" in pos: x = base.width - logo.width - margin
    if "bottom" in pos: y = base.height - logo.height - margin
    out = base.convert("RGBA")
    out.alpha_composite(logo, (x,y))
    return out

def strip_exif_bytes(img: Image.Image) -> Image.Image:
    # Just return img; when saving we’ll drop EXIF by not passing exif data
    return img

def save_image(img: Image.Image, path: str, quality: int, optimize: bool, strip_exif: bool):
    ext = os.path.splitext(path)[1].lower()
    params = {}
    if ext in (".jpg",".jpeg"):
        params.update(dict(quality=quality, subsampling="4:2:0", optimize=optimize, progressive=True))
        if strip_exif:
            # write empty exif
            params["exif"] = piexif.dump({})
        img = img.convert("RGB")
    elif ext == ".png":
        params.update(dict(optimize=optimize))
    img.save(path, **params)

def iter_files(folder: str):
    exts = ("*.jpg","*.jpeg","*.png","*.webp","*.JPG","*.PNG","*.JPEG","*.WEBP")
    files = []
    for e in exts:
        files += sorted(glob.glob(os.path.join(folder, e)))
    return files

def next_name(pattern: Optional[str], index: int, orig_name: str) -> str:
    if not pattern:
        return os.path.splitext(os.path.basename(orig_name))[0]
    # pattern supports {index} and {name}
    base = os.path.splitext(os.path.basename(orig_name))[0]
    try:
        return pattern.format(index=index, name=base)
    except Exception:
        return f"{pattern}_{index:03d}"

def main():
    ap = argparse.ArgumentParser(description="IG Batch Tool: resize/optimize/watermark/rename for Instagram.")
    ap.add_argument("--input", required=True, help="Input folder")
    ap.add_argument("--output", required=True, help="Output folder")
    # Preset or longest
    ap.add_argument("--preset", choices=list(PRESETS.keys()), help="story/post/square exact canvas")
    ap.add_argument("--longest", type=int, default=None, help="If set, resize by longest side instead of preset")
    ap.add_argument("--safe-pad", type=int, default=0, help="Padding inside canvas (for text safe-area)")
    ap.add_argument("--bgcolor", default="#000000", help="Background color for letterbox (hex like #111827)")
    # Logo
    ap.add_argument("--logo", default=None, help="Path to logo PNG (with transparency)")
    ap.add_argument("--logo-pos", choices=["top-left","top-right","bottom-left","bottom-right"], default="bottom-right")
    ap.add_argument("--logo-scale", type=float, default=0.18, help="Logo width relative to image width (0..1)")
    ap.add_argument("--logo-margin", type=int, default=40)
    # Output
    ap.add_argument("--quality", type=int, default=88, help="JPEG quality")
    ap.add_argument("--optimize", action="store_true", help="Optimize on save")
    ap.add_argument("--strip-exif", action="store_true", help="Remove EXIF/metadata")
    ap.add_argument("--ext", choices=["jpg","png"], default="jpg", help="Output extension")
    ap.add_argument("--rename", default=None, help='Rename pattern, e.g. "kafsh_{index:03d}" or "vorojak_{name}"')
    args = ap.parse_args()

    ensure_dir(args.output)
    files = iter_files(args.input)
    if not files:
        print("No images found in --input")
        sys.exit(1)

    # parse bgcolor
    bg = args.bgcolor.lstrip("#")
    if len(bg) == 3: bg = "".join(ch*2 for ch in bg)
    bgcolor = tuple(int(bg[i:i+2],16) for i in (0,2,4))

    W,H = PRESETS.get(args.preset, (None, None))

    count = 0
    for i, fp in enumerate(files, start=1):
        img = load_image(fp)

        if args.longest:
            img = resize_longest(img, args.longest)
            out_img = img
        else:
            if not args.preset:
                print("Set either --preset or --longest")
                sys.exit(1)
            out_img = fit_canvas(img, (W,H), safe_pad=args.safe_pad, bgcolor=bgcolor)

        # logo
        if args.logo:
            out_img = place_logo(out_img, args.logo, args.logo_pos, args.logo_scale, args.logo_margin)

        # build output name
        base = next_name(args.rename, i, fp)
        out_path = os.path.join(args.output, f"{base}.{args.ext}")

        # strip exif (on save)
        save_image(out_img, out_path, quality=args.quality, optimize=args.optimize, strip_exif=args.strip-exif if hasattr(args, "strip-exif") else args.strip_exif)
        count += 1
        print(f"✓ {fp} → {out_path}")

    print(f"\n✅ Done. Processed {count} file(s) into {args.output}")

if __name__ == "__main__":
    main()

