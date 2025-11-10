# Create an animated "glowing serpent" GIF from the generated image
from PIL import Image, ImageEnhance, ImageFilter, ImageOps
import numpy as np

src_path = "/mnt/data/A_digital_illustration_features_a_neon_green_serpe.png"
base = Image.open(src_path).convert("RGBA")

# Scale down for README-friendly size
max_w = 640
scale = min(1.0, max_w / base.width)
if scale < 1.0:
    new_size = (int(base.width*scale), int(base.height*scale))
    base = base.resize(new_size, Image.LANCZOS)

# Prepare frames with a soft neon breathing effect
frames = []
n = 32  # total frames
# Precompute breathing curve (smooth in/out)
t = np.linspace(0, 2*np.pi, n, endpoint=False)
curve = (np.sin(t) + 1) / 2  # 0..1
# Modulate between 85% and 160% brightness
bright_min, bright_max = 0.85, 1.60

# Create a subtle outer glow by repeatedly blurring an expanded alpha
orig = base.copy()
alpha = orig.split()[-1]
glow_color = (0, 255, 140, 255)  # neon green-ish
for i, c in enumerate(curve):
    # Brightness change
    b = bright_min + (bright_max - bright_min) * c
    bright = ImageEnhance.Brightness(orig).enhance(b)
    
    # Outer glow intensity scales with curve
    glow_strength = 6 + int(14 * c)  # blur radius
    # Make glow layer from alpha
    glow = Image.new("RGBA", bright.size, (0, 0, 0, 0))
    # solid neon based on alpha
    solid = Image.new("RGBA", bright.size, glow_color)
    solid.putalpha(alpha)
    glow = Image.alpha_composite(glow, solid)
    # Expand via blur
    for _ in range(2):
        glow = glow.filter(ImageFilter.GaussianBlur(glow_strength))
    # Slightly fade glow to avoid clipping
    glow = ImageEnhance.Brightness(glow).enhance(0.8 + 0.4 * c)
    
    # Combine on black background
    frame = Image.new("RGBA", bright.size, (0, 0, 0, 255))
    frame = Image.alpha_composite(frame, glow)
    frame = Image.alpha_composite(frame, bright)
    
    # Subtle vignette
    vignette = Image.new("L", frame.size, 0)
    # radial gradient
    w, h = frame.size
    x = np.linspace(-1, 1, w)[None, :]
    y = np.linspace(-1, 1, h)[:, None]
    r = np.sqrt(x**2 + y**2)
    grad = (1 - np.clip((r - 0.3) / 0.7, 0, 1)) * 220  # 0..220
    vignette = Image.fromarray(grad.astype(np.uint8), mode="L")
    vignette = vignette.filter(ImageFilter.GaussianBlur(8))
    frame.putalpha(255)
    frame = Image.composite(frame, Image.new("RGBA", frame.size, (0,0,0,255)), ImageOps.invert(vignette))
    frame = frame.convert("P", palette=Image.ADAPTIVE)
    frames.append(frame)

out_path = "/mnt/data/serpent_glow.gif"
# Save as looping GIF (≈ 24 fps -> 42ms/frame). Keep it sub-2MB typically.
frames[0].save(out_path, save_all=True, append_images=frames[1:], duration=42, loop=0, optimize=True)

out_path
