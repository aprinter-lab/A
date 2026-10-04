# -*- coding: utf-8 -*-
"""
旋转星球 —— 用纯标准库 (tkinter) 实现的 3D 旋转星球动画。
 
无需安装任何第三方库，直接运行即可：
 
    python rotating_planet.py
 
操作：
    空格       暂停 / 继续
    ↑ / ↓      加快 / 减慢转速（可反向旋转）
    R          随机生成一颗新星球
    Esc        退出
"""
 
import math
import random
import tkinter as tk
 
# ================= 配置 =================
WIDTH, HEIGHT = 900, 640
CX, CY = WIDTH // 2, HEIGHT // 2
RADIUS = 200
 
SAMPLE = 4200                # 球面采样点数（用来生成大陆纹理）
TILT = math.radians(23.5)    # 地轴倾角
LIGHT = (0.42, 0.55, 0.72)   # 光源方向（会被归一化）
SHADE_LEVELS = 24            # 明暗查找表分级数
FRAME_MS = 40                # 帧间隔（毫秒）
BASE_STEPS = 90              # 星球底色同心圆层数
ROT_STEP = 0.010             # 每帧基础旋转弧度
 
# 大陆中心（纬度, 经度），单位：度
CONTINENTS = [
    (55, -100),   # 北美
    (-15, -60),   # 南美
    (50, 15),     # 欧洲
    (8, 20),      # 非洲
    (45, 90),     # 亚洲
    (-25, 135),   # 澳洲
    (60, 115),    # 西伯利亚
]
 
 
# ================= 工具 =================
def clamp(x, lo, hi):
    if x < lo:
        return lo
    if x > hi:
        return hi
    return x
 
 
def mix(c1, c2, t):
    return (c1[0] + (c2[0] - c1[0]) * t,
            c1[1] + (c2[1] - c1[1]) * t,
            c1[2] + (c2[2] - c1[2]) * t)
 
 
def smoothstep(a, b, x):
    t = clamp((x - a) / (b - a), 0.0, 1.0)
    return t * t * (3.0 - 2.0 * t)
 
 
def hexc(c):
    r = int(clamp(c[0], 0, 255))
    g = int(clamp(c[1], 0, 255))
    b = int(clamp(c[2], 0, 255))
    return "#%02x%02x%02x" % (r, g, b)
 
 
# 归一化光源
_lm = math.sqrt(LIGHT[0] ** 2 + LIGHT[1] ** 2 + LIGHT[2] ** 2)
LX, LY, LZ = LIGHT[0] / _lm, LIGHT[1] / _lm, LIGHT[2] / _lm
 
 
def _continent_vectors():
    vecs = []
    for lat, lon in CONTINENTS:
        la, lo = math.radians(lat), math.radians(lon)
        vecs.append((math.cos(la) * math.cos(lo),
                     math.sin(la),
                     math.cos(la) * math.sin(lo)))
    return vecs
 
 
# ================= 生成星球 =================
def build_planet():
    """在单位球上采样，返回用于渲染的（陆地和冰盖）点列表。"""
    cont = _continent_vectors()
    cosT, sinT = math.cos(TILT), math.sin(TILT)
    golden = math.pi * (3.0 - math.sqrt(5.0))
 
    pts = []
    for i in range(SAMPLE):
        yy = 1.0 - (i / (SAMPLE - 1)) * 2.0
        rr = math.sqrt(max(0.0, 1.0 - yy * yy))
        th = golden * i
        x = math.cos(th) * rr
        z = math.sin(th) * rr
        lat = math.asin(yy)
 
        # 大陆评分：到各大陆中心的高斯累加
        score = 0.0
        for cx0, cy0, cz0 in cont:
            d = clamp(x * cx0 + yy * cy0 + z * cz0, -1.0, 1.0)
            ang = math.acos(d)
            score += math.exp(-(ang * ang) / (2.0 * 0.34 * 0.34))
        noise = random.random()
        is_land = score > (0.55 + (noise - 0.5) * 0.12)
 
        ice = smoothstep(1.05, 1.42, abs(lat))
        if ice < 1e-6 and not is_land:
            continue  # 纯海洋不渲染，交给静态底色
 
        # 基础颜色
        if is_land:
            if abs(lat) > 0.95:
                base = mix((150, 145, 130), (120, 118, 100), random.random())
            else:
                base = mix((64, 122, 60), (155, 118, 66), random.random())
        else:
            base = mix((14, 44, 88), (40, 96, 150), random.random())
 
        # 冰盖混合
        if ice > 0.0:
            base = mix(base, (238, 244, 250), ice)
 
        # 轻微亮度抖动，增加质感
        jitter = 0.90 + random.random() * 0.20
        base = (base[0] * jitter, base[1] * jitter, base[2] * jitter)
 
        # 绕 X 轴倾斜（地轴）
        y2 = yy * cosT - z * sinT
        z2 = yy * sinT + z * cosT
        pts.append([x, y2, z2, base])
 
    return pts
 
 
def build_lut(pts):
    """为每个点预生成不同亮度下的颜色，避免逐帧做颜色运算。"""
    L = SHADE_LEVELS
    lut = []
    for _, _, _, (r, g, b) in pts:
        arr = []
        for k in range(L):
            f = k / (L - 1)
            arr.append(hexc((r * f, g * f, b * f)))
        lut.append(arr)
    return lut
 
 
# ================= 星空 =================
def make_stars():
    stars = []
    for _ in range(200):
        s = random.uniform(0.4, 1.6)
        b = random.uniform(0.35, 1.0)
        c = int(180 * b + 40)
        tint = random.choice([
            (c, c, c),
            (c, int(c * 0.88), int(c * 0.72)),
            (int(c * 0.78), int(c * 0.86), c),
        ])
        stars.append([random.uniform(0, WIDTH), random.uniform(0, HEIGHT), s, tint])
    return stars
 
 
# ================= 绘制 =================
def draw_base():
    """静态星球底色：偏置圆心的同心圆堆叠，形成立体光影。"""
    canvas.delete("base")
    for i in range(BASE_STEPS):
        t = i / (BASE_STEPS - 1)
        rr = RADIUS * (1.0 - t * 0.99)
        off = RADIUS * 0.30 * t
        ox = CX + off * 0.60   # 高光偏向右上，与光源一致
        oy = CY - off * 0.80
        c = mix((14, 40, 84), (96, 156, 206), (1.0 - t) ** 1.35)
        col = hexc(c)
        canvas.create_oval(ox - rr, oy - rr, ox + rr, oy + rr,
                           fill=col, outline=col, tags="base")
 
 
def draw_ring():
    canvas.delete("ring")
    canvas.create_oval(CX - RADIUS - 1, CY - RADIUS - 1,
                       CX + RADIUS + 1, CY + RADIUS + 1,
                       outline="#3d5f85", width=2, tags="ring")
    canvas.create_oval(CX - RADIUS - 6, CY - RADIUS - 6,
                       CX + RADIUS + 6, CY + RADIUS + 6,
                       outline="#33557f", width=3, stipple="gray50", tags="ring")
    canvas.create_oval(CX - RADIUS - 14, CY - RADIUS - 14,
                       CX + RADIUS + 14, CY + RADIUS + 14,
                       outline="#2a4a72", width=2, stipple="gray25", tags="ring")
 
 
def draw_stars():
    global star_items
    canvas.delete("star")
    star_items = []
    for sx, sy, s, tint in stars:
        it = canvas.create_oval(sx - s, sy - s, sx + s, sy + s,
                                fill=hexc(tint), outline="", tags="star")
        star_items.append(it)
 
 
def twinkle():
    global twinkle_tick
    twinkle_tick += 1
    if twinkle_tick % 4 != 0:
        return
    for _ in range(12):
        it = random.choice(star_items)
        b = random.uniform(0.3, 1.0)
        c = int(180 * b + 40)
        canvas.itemconfig(it, fill=hexc((c, c, c)))
 
 
def frame():
    if not running:
        return
    global angle
    angle += speed * ROT_STEP
    ca, sa = math.cos(angle), math.sin(angle)
    Lm = SHADE_LEVELS - 1
    canvas.delete("planet")
 
    for i, (x, y, z, _) in enumerate(pts):
        nx = x * ca + z * sa
        nz = -x * sa + z * ca
        if nz <= 0.02:
            continue
        ny = y
        b = LX * nx + LY * ny + LZ * nz
        b = 0.30 + 0.70 * max(0.0, b)
        rim = 0.55 + 0.45 * nz
        b *= rim
        idx = int(b * Lm + 0.5)
        idx = clamp(idx, 0, Lm)
        fill = lut[i][idx]
        sx = CX + RADIUS * nx
        sy = CY - RADIUS * ny
        s = 1.6 + 1.0 * nz
        canvas.create_oval(sx - s, sy - s, sx + s, sy + s,
                           fill=fill, outline=fill, tags="planet")
 
    twinkle()
    canvas.tag_raise("ring")
    root.after(FRAME_MS, frame)
 
 
def rebuild():
    global pts, lut
    pts = build_planet()
    lut = build_lut(pts)
    draw_base()
    canvas.tag_raise("ring")
 
 
def on_key(e):
    global running, speed
    k = e.keysym
    if k == "space":
        running = not running
        if running:
            frame()
    elif k == "Up":
        speed = min(speed + 1, 8)
    elif k == "Down":
        speed = max(speed - 1, -8)
    elif k.lower() == "r":
        rebuild()
    elif k == "Escape":
        running = False
        root.destroy()
 
 
def on_close():
    global running
    running = False
    root.destroy()
 
 
# ================= 主程序 =================
if __name__ == "__main__":
    random.seed(20261003)
 
    root = tk.Tk()
    root.title("旋转星球 Rotating Planet")
    canvas = tk.Canvas(root, width=WIDTH, height=HEIGHT,
                       bg="#04040d", highlightthickness=0)
    canvas.pack()
 
    angle = 0.0
    speed = 1
    running = True
    twinkle_tick = 0
 
    stars = make_stars()
    star_items = []
    pts = build_planet()
    lut = build_lut(pts)
 
    draw_stars()
    draw_base()
    draw_ring()
 
    canvas.create_text(CX, 40, text="旋转星球",
                       fill="#cfe4ff", font=("Microsoft YaHei", 30, "bold"),
                       tags="title")
    canvas.create_text(CX, HEIGHT - 28,
                       text="空格 暂停 · ↑/↓ 调速 · R 换星球 · Esc 退出",
                       fill="#5a7a9a", font=("Microsoft YaHei", 11),
                       tags="title")
    canvas.tag_raise("title")
 
    root.bind("<KeyPress>", on_key)
    root.protocol("WM_DELETE_WINDOW", on_close)
 
    frame()
    root.mainloop()
