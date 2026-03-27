"""
AI Data Center — Water & Thermal Simulation. 24-hour simulation of a data center's heat output and water usage, based on server workloads and energy mix.

NOTE: This is my attempt at simulating water usage in data centers.
I'm still tweaking the formulas, so don't take these numbers too seriously!
"""

import tkinter as tk
from tkinter import ttk
import numpy as np
import matplotlib
matplotlib.use("TkAgg")
import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
from matplotlib.colors import LinearSegmentedColormap
import matplotlib.patheffects as pe
import random

# Color scheme - Most of these are just for styling the charts and UI, but I kept the heat/water colors consistent in the charts and stat cards
BG_PAGE  = "#0b0d11"
BG_CARD  = "#0f1115"
BG_CHART = "#0d0f13"
BORDER   = "#1e2330"
MUTED    = "#6b7a99"
TEXT     = "#c8d4e0"
DIM      = "#8a9ab8"

C_HEAT    = "#FF4500"  # orangered for heat - stands out well against the dark background
C_WATER   = "#1a7fd4"
C_OFFSITE = "#28a844"
C_ENERGY  = "#e8d87a"
C_ACCENT  = "#a076d4"

# Data center config - based on some rough estimates I found online
SERVERS           = 50
SERVER_WATTS      = 400  # per server
TOKENS_PER_SERVER = 1_000_000  # tokens processed per hour

# Energy mix percentages (share, carbon factor)
# TODO: Maybe make this configurable later?
ENERGY_MIX = [
    ("Coal",  0.30, 1.90),
    ("Gas",   0.40, 0.90),
    ("Solar", 0.20, 0.05),
    ("Wind",  0.10, 0.01),
]
OFFSITE_FACTOR = sum(s * f for _, s, f in ENERGY_MIX)


def noise(pct=0.10):
    """Added some variation to make it look more realistic, just like how real data centers aren't perfectly consistent hour to hour"""
    return 1.0 + random.uniform(-pct, pct)


def rand(lo, hi):
    """Just a shorthand because I use this a lot for generating random parameters and values within a range"""
    return random.uniform(lo, hi)


# Generate simulation data
def generate_simulation():
    # These values should vary between runs
    utilization = rand(0.85, 0.95)  # servers don't run at 100% usually
    PUE         = rand(1.10, 1.30)  # Power Usage Effectiveness
    WUE         = rand(2.00, 3.00)  # Water Usage Effectiveness (liters per kWh)

    hours = []
    
    # Loop through 24 hours
    for h in range(24):
        # Calculate IT load in kilowatts
        it_kw       = SERVERS * SERVER_WATTS / 1_000 * utilization
        
        # Total energy including cooling overhead
        energy_kwh  = it_kw * PUE * noise()
        
        # Heat calculations - converting energy to joules
        heat_j      = energy_kwh * 3_600_000 * noise()  # 1 kWh = 3.6M joules
        heat_w      = (heat_j / 3_600) * noise()  # back to watts for display
        
        # Water usage - on-site cooling tower + off-site from power generation
        onsite_L    = energy_kwh * WUE            * noise()
        offsite_L   = energy_kwh * OFFSITE_FACTOR * noise()
        total_water = onsite_L + offsite_L
        
        # Token processing
        tokens      = SERVERS * TOKENS_PER_SERVER

        hours.append({
            "hour"        : h,
            "label"       : f"{h:02d}:00",
            "energy_kwh"  : energy_kwh,
            "heat_j"      : heat_j,
            "heat_w"      : heat_w,
            "onsite_L"    : onsite_L,
            "offsite_L"   : offsite_L,
            "total_water" : total_water,
            "tokens"      : tokens,
        })

    # Calculate totals for the summary
    te  = sum(r["energy_kwh"]  for r in hours)
    thj = sum(r["heat_j"]      for r in hours)
    ton = sum(r["onsite_L"]    for r in hours)
    tof = sum(r["offsite_L"]   for r in hours)
    tw  = sum(r["total_water"] for r in hours)
    tt  = sum(r["tokens"]      for r in hours)

    summary = {
        "total_energy_kwh"   : te,
        "total_heat_gj"      : thj / 1e9,  # gigajoules looks better than joules and is more comparable to other energy figures
        "total_heat_j"       : thj,
        "total_onsite_L"     : ton,
        "total_offsite_L"    : tof,
        "total_water_L"      : tw,
        "water_per_token_mL" : (tw / tt) * 1_000,
        "heat_per_token_j"   : thj / tt,
        "total_tokens"       : tt,
    }

    params = {
        "servers"     : SERVERS,
        "PUE"         : PUE,
        "WUE"         : WUE,
        "utilization" : utilization,
    }

    return hours, summary, params


# Helper to style chart axes consistently and make it look a bit nicer overall
def style_ax(ax, title="", ylabel="", grid=True):
    ax.set_facecolor(BG_CHART)
    ax.tick_params(colors=MUTED, labelsize=7.5, length=3)
    
    # Set border colors
    for spine in ax.spines.values():
        spine.set_edgecolor(BORDER)
        
    if title:
        ax.set_title(title, color=DIM, fontsize=8, fontfamily="monospace",
                     pad=7, fontweight="bold")
    if ylabel:
        ax.set_ylabel(ylabel, color=MUTED, fontsize=7.5, fontfamily="monospace")
        
    ax.tick_params(axis="x", colors=MUTED)
    ax.tick_params(axis="y", colors=MUTED)
    
    if grid:
        # Y-axis grid more prominent than X
        ax.grid(axis="y", color=BORDER, linewidth=0.6, linestyle="--", alpha=0.6)
        ax.grid(axis="x", color=BORDER, linewidth=0.4, linestyle=":",  alpha=0.4)


def gradient_fill(ax, x, y, color, alpha_top=0.50, alpha_bot=0.02):
    """Creates a gradient fill under a line chart - looks nicer than a solid color and helps indicate intensity"""
    from matplotlib.patches import PathPatch
    from matplotlib.path import Path

    # Create polygon vertices
    verts = list(zip(x, y)) + [(x[-1], 0), (x[0], 0)]
    codes = [Path.MOVETO] + [Path.LINETO] * (len(x) - 1) + [Path.LINETO, Path.LINETO]
    patch = PathPatch(Path(verts, codes), facecolor="none", edgecolor="none")
    ax.add_patch(patch)

    # Build gradient colormap
    import matplotlib.colors as mc
    rgba_top = list(mc.to_rgba(color))
    rgba_top[3] = alpha_top
    rgba_bot = list(mc.to_rgba(color))
    rgba_bot[3] = alpha_bot
    cmap = LinearSegmentedColormap.from_list("g", [rgba_bot, rgba_top])

    # Apply gradient as image
    ax.imshow(
        np.linspace(0, 1, 256).reshape(256, 1),
        aspect="auto",
        extent=[min(x), max(x), 0, max(y) * 1.05],
        origin="lower", cmap=cmap,
        clip_path=patch, clip_on=True, zorder=0,
    )


# Hover tooltip implementation - shows data when you hover over a chart point
class HoverTooltip:
    def __init__(self, fig, ax, mpl_canvas, hours, fields):
        self.ax       = ax
        self.hours    = hours
        self.fields   = fields
        self._win     = None

        # Connect mouse events
        mpl_canvas.mpl_connect("motion_notify_event", self._on_move)
        mpl_canvas.mpl_connect("axes_leave_event",    self._hide)

    def _on_move(self, event):
        if event.inaxes != self.ax or event.xdata is None:
            self._hide(None)
            return
            
        # Find closest hour
        idx = int(round(event.xdata))
        if 0 <= idx < len(self.hours):
            self._show(self.hours[idx])
        else:
            self._hide(None)

    def _show(self, r):
        # Destroy old tooltip if exists
        if self._win:
            self._win.destroy()

        # Create new tooltip window
        win = tk.Toplevel()
        win.overrideredirect(True)  # no window decorations
        win.attributes("-topmost", True)
        win.configure(bg="#0e1320",
                      highlightbackground="#2a3655",
                      highlightthickness=1)

        # Hour header
        tk.Label(win, text=f"  {r['label']}",
                 bg="#0e1320", fg=MUTED,
                 font=("Courier New", 8, "bold"),
                 pady=6).pack(anchor="w")

        tk.Frame(win, bg=BORDER, height=1).pack(fill="x")

        # Data rows
        for label, key, fmt in self.fields:
            row = tk.Frame(win, bg="#0e1320")
            row.pack(fill="x", padx=10, pady=2)
            tk.Label(row, text=label + ":",
                     bg="#0e1320", fg=MUTED,
                     font=("Courier New", 8)).pack(side="left")
            tk.Label(row, text=fmt(r[key]),
                     bg="#0e1320", fg=TEXT,
                     font=("Courier New", 8, "bold")).pack(side="right", padx=(12, 0))

        tk.Frame(win, bg="#0e1320", height=6).pack()

        # Position near cursor
        win.update_idletasks()
        sx = win.winfo_screenwidth()
        sy = win.winfo_screenheight()
        cx = win.winfo_pointerx()
        cy = win.winfo_pointery()
        
        # Make sure tooltip stays on screen since some of these charts are near the edges
        tx = min(cx + 14, sx - win.winfo_reqwidth()  - 8)
        ty = min(cy + 14, sy - win.winfo_reqheight() - 8)
        win.geometry(f"+{tx}+{ty}")
        self._win = win

    def _hide(self, event):
        if self._win:
            self._win.destroy()
            self._win = None


# Main dashboard class
class Dashboard:
    def __init__(self, root):
        self.root      = root
        self.root.title("AI Data Center — Water & Thermal Simulation")
        self.root.configure(bg=BG_PAGE)
        self.root.state("zoomed")  # start maximized
        self._tooltips = []

        self._build_header()
        self._build_scroll_body()
        self.simulate()  # run initial simulation

    def _build_header(self):
        """Top header bar with title and re-simulate button"""
        hdr = tk.Frame(self.root, bg="#090b0e", pady=11, padx=20)
        hdr.pack(fill="x", side="top")

        left = tk.Frame(hdr, bg="#090b0e")
        left.pack(side="left")
        tk.Label(left, text="AI Data Center",
                 bg="#090b0e", fg=C_ENERGY,
                 font=("Courier New", 13, "bold")).pack(anchor="w")
        tk.Label(left, text="Water consumption & thermal output  —  24-hour simulation",
                 bg="#090b0e", fg=MUTED,
                 font=("Courier New", 8)).pack(anchor="w")

        # Re-simulate button, randomizes the data again when clicked so that you can see different variations and test the tooltips on the charts :)
        self.btn = tk.Button(
            hdr, text="Re-simulate",
            command=self.simulate,
            bg="#151a28", fg=C_ENERGY,
            activebackground="#1e2438", activeforeground=C_ENERGY,
            font=("Courier New", 10, "bold"),
            relief="flat", bd=0, padx=16, pady=7, cursor="hand2",
        )
        self.btn.pack(side="right")

    def _build_scroll_body(self):
        """Scrollable main content area"""
        container = tk.Frame(self.root, bg=BG_PAGE)
        container.pack(fill="both", expand=True)

        canvas = tk.Canvas(container, bg=BG_PAGE, highlightthickness=0)
        vsb    = ttk.Scrollbar(container, orient="vertical", command=canvas.yview)
        canvas.configure(yscrollcommand=vsb.set)
        vsb.pack(side="right", fill="y")
        canvas.pack(side="left", fill="both", expand=True)

        self.body     = tk.Frame(canvas, bg=BG_PAGE)
        self.body_win = canvas.create_window((0, 0), window=self.body, anchor="nw")

        def _resize(e):
            canvas.configure(scrollregion=canvas.bbox("all"))
            canvas.itemconfig(self.body_win, width=canvas.winfo_width())

        self.body.bind("<Configure>", _resize)
        canvas.bind("<Configure>",
                    lambda e: canvas.itemconfig(self.body_win, width=e.width))
        # Mouse wheel scrolling
        canvas.bind_all("<MouseWheel>",
                        lambda e: canvas.yview_scroll(int(-1*(e.delta/120)), "units"))

    def simulate(self):
        """Generate new simulation data and update UI"""
        # Disable button while generating
        self.btn.config(state="disabled", text="  generating…")
        self.root.update_idletasks()

        # Clear old content
        for w in self.body.winfo_children():
            w.destroy()
        self._tooltips.clear()

        # Generate fresh data
        hours, summary, params = generate_simulation()
        
        # Build UI components
        self._stat_cards(summary)
        self._charts(hours, summary, params)
        self._table(hours)

        # Re-enable button
        self.btn.config(state="normal", text="↺  Re-simulate")

    def _stat_cards(self, s):
        """Top row of summary stat cards"""
        frame = tk.Frame(self.body, bg=BG_PAGE, padx=18, pady=10)
        frame.pack(fill="x")

        cards = [
            ("Total energy",    f"{s['total_energy_kwh']:.2f}",   "kWh",  C_ENERGY),
            ("Total heat",      f"{s['total_heat_gj']:.4f}",      "GJ",   C_HEAT),
            ("On-site water",   f"{s['total_onsite_L']:.1f}",     "L",    C_WATER),
            ("Total water",     f"{s['total_water_L']:.1f}",      "L",    C_WATER),
            ("Water per token", f"{s['water_per_token_mL']:.4f}", "mL",   C_OFFSITE),
            ("Heat per token",  f"{s['heat_per_token_j']:.5f}",   "J",    C_HEAT),
        ]

        for col, (label, value, unit, color) in enumerate(cards):
            card = tk.Frame(frame, bg=BG_CARD,
                            highlightbackground=BORDER,
                            highlightthickness=1,
                            padx=14, pady=12)
            card.grid(row=0, column=col, padx=5, pady=2, sticky="nsew")
            frame.columnconfigure(col, weight=1)

            tk.Label(card, text=label,
                     bg=BG_CARD, fg=MUTED,
                     font=("Courier New", 8)).pack(anchor="w")
            tk.Label(card, text=value,
                     bg=BG_CARD, fg=color,
                     font=("Courier New", 15, "bold")).pack(anchor="w", pady=(3, 0))
            tk.Label(card, text=unit,
                     bg=BG_CARD, fg=DIM,
                     font=("Courier New", 8)).pack(anchor="w")

    def _charts(self, hours, summary, params):
        """Generate all the visualization charts"""
        # Extract data arrays for plotting
        xs   = [r["hour"]       for r in hours]
        lbls = [r["label"]      for r in hours]
        hw   = [r["heat_w"]     for r in hours]
        on   = [r["onsite_L"]   for r in hours]
        off  = [r["offsite_L"]  for r in hours]
        en   = [r["energy_kwh"] for r in hours]

        # Set matplotlib style
        plt.rcParams.update({
            "figure.facecolor": BG_PAGE,
            "axes.facecolor"  : BG_CHART,
            "text.color"      : TEXT,
            "font.family"     : "monospace",
        })

        # First row: Heat area chart + Stacked water bar chart
        fig1, (ax_heat, ax_water) = plt.subplots(
            1, 2, figsize=(14, 3.4), facecolor=BG_PAGE,
            gridspec_kw={"wspace": 0.30})

        style_ax(ax_heat, title="Heat output (W)")
        ax_heat.plot(xs, hw, color=C_HEAT, linewidth=2, zorder=3,
                     path_effects=[pe.Stroke(linewidth=3.5,
                                             foreground="#ff450018"), pe.Normal()])
        gradient_fill(ax_heat, xs, hw, C_HEAT)
        ax_heat.set_xlim(0, 23)
        ax_heat.set_xticks(xs[::2])  # show every other hour
        ax_heat.set_xticklabels(lbls[::2], rotation=45, ha="right", fontsize=7)

        style_ax(ax_water, title="Water consumption (L/h)")
        b1 = ax_water.bar(xs, on,  width=0.72, color=C_WATER,   alpha=0.85, label="On-site")
        b2 = ax_water.bar(xs, off, width=0.72, color=C_OFFSITE, alpha=0.85,
                          bottom=on, label="Off-site")
        ax_water.legend(handles=[b1, b2], facecolor=BG_CARD, edgecolor=BORDER,
                        labelcolor=TEXT, fontsize=7, loc="upper right")
        ax_water.set_xticks(xs[::2])
        ax_water.set_xticklabels(lbls[::2], rotation=45, ha="right", fontsize=7)

        # Attach tooltips to charts
        self._embed(fig1, hours,
            axes_list   = [ax_heat, ax_water],
            fields_list = [
                [
                    ("Hour",        "label",      lambda v: v),
                    ("Heat",        "heat_w",     lambda v: f"{v/1000:.3f} kW"),
                    ("Heat total",  "heat_j",     lambda v: f"{v/1e6:.3f} MJ"),
                    ("Energy",      "energy_kwh", lambda v: f"{v:.3f} kWh"),
                ],
                [
                    ("Hour",        "label",      lambda v: v),
                    ("On-site",     "onsite_L",   lambda v: f"{v:.2f} L"),
                    ("Off-site",    "offsite_L",  lambda v: f"{v:.2f} L"),
                    ("Total water", "total_water",lambda v: f"{v:.2f} L"),
                    ("Energy",      "energy_kwh", lambda v: f"{v:.3f} kWh"),
                ],
            ]
        )

        # Second row: Energy line chart + On/off comparison bar
        fig2 = plt.figure(figsize=(14, 3.2), facecolor=BG_PAGE)
        gs   = gridspec.GridSpec(1, 3, figure=fig2, wspace=0.32)
        ax_en  = fig2.add_subplot(gs[0, :2])
        ax_cmp = fig2.add_subplot(gs[0, 2])

        style_ax(ax_en, title="Energy consumption (kWh)")
        ax_en.plot(xs, en, color=C_ENERGY, linewidth=2.2,
                   marker="o", markersize=3.5,
                   markerfacecolor=C_ENERGY, zorder=3)
        gradient_fill(ax_en, xs, en, C_ENERGY, alpha_top=0.32)
        ax_en.set_xlim(0, 23)
        ax_en.set_xticks(xs[::2])
        ax_en.set_xticklabels(lbls[::2], rotation=45, ha="right", fontsize=7)

        style_ax(ax_cmp, title="On-site vs off-site (total L)")
        cats = ["On-site", "Off-site"]
        vals = [summary["total_onsite_L"], summary["total_offsite_L"]]
        bars = ax_cmp.bar(cats, vals, color=[C_WATER, C_OFFSITE], alpha=0.88, width=0.5)
        
        # Add value labels on bars
        for bar, val in zip(bars, vals):
            ax_cmp.text(bar.get_x() + bar.get_width() / 2,
                        bar.get_height() + max(vals) * 0.02,
                        f"{val:.1f}", ha="center", va="bottom",
                        color=TEXT, fontsize=8, fontfamily="monospace")
        ax_cmp.set_xticklabels(cats, color=TEXT, fontsize=8)

        self._embed(fig2, hours,
            axes_list   = [ax_en],
            fields_list = [
                [
                    ("Hour",     "label",      lambda v: v),
                    ("Energy",   "energy_kwh", lambda v: f"{v:.3f} kWh"),
                    ("On-site",  "onsite_L",   lambda v: f"{v:.2f} L"),
                    ("Off-site", "offsite_L",  lambda v: f"{v:.2f} L"),
                    ("Heat",     "heat_w",     lambda v: f"{v/1000:.3f} kW"),
                ],
            ]
        )

        # Third row: Thermal intensity gauge + Parameters panel
        fig3, (ax_th, ax_pr) = plt.subplots(
            1, 2, figsize=(14, 2.6), facecolor=BG_PAGE,
            gridspec_kw={"wspace": 0.32})

        style_ax(ax_th, title="Thermal intensity — heat per token (J)", grid=False)
        val_norm = min(summary["heat_per_token_j"] / 0.15, 1.0)  # normalize to 0-1
        ax_th.set_xlim(0, 1)
        ax_th.set_ylim(-0.5, 0.5)
        ax_th.set_yticks([])
        
        # Background bar
        ax_th.barh(0, 1, height=0.25, color=BORDER, left=0, zorder=1)
        
        # Gradient fill bar
        grad = LinearSegmentedColormap.from_list("hg", ["#ff7700", C_HEAT, "#bb0000"])
        for i in np.linspace(0, val_norm, 300):
            ax_th.barh(0, 1/300, height=0.25,
                       color=grad(i / max(val_norm, 0.01)),
                       left=i, zorder=2, alpha=0.92)
        
        # Current value indicator
        ax_th.axvline(val_norm, color="white", linewidth=1.4,
                      ymin=0.32, ymax=0.68, zorder=3)
        ax_th.text(val_norm + 0.02, 0,
                   f"{summary['heat_per_token_j']:.5f} J",
                   va="center", color=C_HEAT,
                   fontsize=9, fontfamily="monospace", fontweight="bold")
        ax_th.set_xticks([0, 0.25, 0.5, 0.75, 1.0])
        ax_th.set_xticklabels(["0", "25%", "50%", "75%", "100%"],
                               color=MUTED, fontsize=7.5)
        for sp in ax_th.spines.values():
            sp.set_visible(False)

        # Parameters info panel
        ax_pr.set_facecolor(BG_CARD)
        for sp in ax_pr.spines.values():
            sp.set_edgecolor(BORDER)
        ax_pr.set_xticks([])
        ax_pr.set_yticks([])
        ax_pr.set_title("Simulation parameters", color=DIM, fontsize=8,
                         fontfamily="monospace", pad=7, fontweight="bold")

        param_rows = [
            ("Servers",       str(params["servers"]),             TEXT),
            ("Watts/server",  f"{SERVER_WATTS} W",                TEXT),
            ("Utilization",   f"{params['utilization']:.4f}",     C_ENERGY),
            ("PUE",           f"{params['PUE']:.4f}",             C_ACCENT),
            ("WUE",           f"{params['WUE']:.4f} L/kWh",      C_WATER),
            ("Tokens/server", f"{TOKENS_PER_SERVER:,}",           C_OFFSITE),
            ("Total tokens",  f"{summary['total_tokens']:,.0f}",  C_OFFSITE),
            ("Energy mix",    "30% coal  40% gas",                MUTED),
            ("",              "20% solar  10% wind",              MUTED),
        ]

        for i, (k, v, c) in enumerate(param_rows):
            y = 0.90 - i * 0.105
            if k:
                ax_pr.text(0.04, y, k + ":", transform=ax_pr.transAxes,
                           color=MUTED, fontsize=8, fontfamily="monospace", va="top")
            ax_pr.text(0.97, y, v, transform=ax_pr.transAxes,
                       color=c, fontsize=8.5, fontfamily="monospace",
                       va="top", ha="right", fontweight="bold")

        self._embed(fig3, hours, axes_list=[], fields_list=[])

    def _embed(self, fig, hours, axes_list, fields_list):
        """Embed matplotlib figure into tkinter and setup tooltips"""
        frame  = tk.Frame(self.body, bg=BG_PAGE, padx=18, pady=5)
        frame.pack(fill="x")
        mpl_canvas = FigureCanvasTkAgg(fig, master=frame)
        mpl_canvas.draw()
        mpl_canvas.get_tk_widget().pack(fill="x")

        # Attach tooltip to each axis
        for ax, fields in zip(axes_list, fields_list):
            if fields:
                tt = HoverTooltip(fig, ax, mpl_canvas, hours, fields)
                self._tooltips.append(tt)

        plt.close(fig)

    def _table(self, hours):
        """Bottom table showing hourly breakdown"""
        outer = tk.Frame(self.body, bg=BG_PAGE, padx=18, pady=8)
        outer.pack(fill="x")

        tk.Label(outer, text="Hourly breakdown",
                 bg=BG_PAGE, fg=DIM,
                 font=("Courier New", 9, "bold")).pack(anchor="w", pady=(0, 5))

        tframe = tk.Frame(outer, bg=BG_CARD,
                          highlightbackground=BORDER, highlightthickness=1)
        tframe.pack(fill="x")

        hscroll = tk.Scrollbar(tframe, orient="horizontal")
        hscroll.pack(side="bottom", fill="x")

       
        style = ttk.Style()
        style.theme_use("clam")
        style.configure("D.Treeview",
                        background=BG_CARD, fieldbackground=BG_CARD,
                        foreground=TEXT, rowheight=21,
                        font=("Courier New", 8))
        style.configure("D.Treeview.Heading",
                        background="#0f1420", foreground=MUTED,
                        relief="flat", font=("Courier New", 8, "bold"))
        style.map("D.Treeview",
                  background=[("selected", "#1a2540")],
                  foreground=[("selected", TEXT)])

        cols = [
            ("Hour",            60),
            ("Energy (kWh)",    95),
            ("Heat (MJ)",       90),
            ("Heat (kW)",       90),
            ("On-site (L)",     90),
            ("Off-site (L)",    90),
            ("Total water (L)", 105),
            ("Tokens",          90),
        ]

        tree = ttk.Treeview(
            tframe, style="D.Treeview",
            columns=[c[0] for c in cols],
            show="headings", height=24,
            xscrollcommand=hscroll.set,
        )

        for name, width in cols:
            tree.heading(name, text=name)
            tree.column(name,
                        anchor="center" if name == "Hour" else "e",
                        width=width, minwidth=width)

        # Alternating row colors
        tree.tag_configure("odd",  background="#0c1014")
        tree.tag_configure("even", background=BG_CARD)

        for r in hours:
            tag = "odd" if r["hour"] % 2 else "even"
            tree.insert("", "end", tags=(tag,), values=(
                r["label"],
                f"{r['energy_kwh']:.3f}",
                f"{r['heat_j'] / 1e6:.3f}",
                f"{r['heat_w'] / 1e3:.3f}",
                f"{r['onsite_L']:.2f}",
                f"{r['offsite_L']:.2f}",
                f"{r['total_water']:.2f}",
                f"{r['tokens']:,}",
            ))

        hscroll.config(command=tree.xview)
        tree.pack(fill="x")

        # Totals footer row
        tot = tk.Frame(outer, bg="#0f1420",
                       highlightbackground=BORDER, highlightthickness=1,
                       padx=10, pady=7)
        tot.pack(fill="x", pady=(2, 8))

        totals = [
            (f"Energy: {sum(r['energy_kwh'] for r in hours):.2f} kWh",      C_ENERGY),
            (f"Heat: {sum(r['heat_j'] for r in hours)/1e6:.2f} MJ",          C_HEAT),
            (f"On-site: {sum(r['onsite_L'] for r in hours):.1f} L",          C_WATER),
            (f"Off-site: {sum(r['offsite_L'] for r in hours):.1f} L",        C_OFFSITE),
            (f"Total water: {sum(r['total_water'] for r in hours):.1f} L",   C_WATER),
            (f"Tokens: {sum(r['tokens'] for r in hours)/1e9:.2f}B",          MUTED),
        ]

        for i, (text, color) in enumerate(totals):
            tk.Label(tot, text=text,
                     bg="#0f1420", fg=color,
                     font=("Courier New", 8, "bold")).grid(
                row=0, column=i, padx=10, sticky="w")


# in order to Run the app
if __name__ == "__main__":
    root = tk.Tk()
    root.minsize(1100, 700)

    # Configure scrollbar style
    s = ttk.Style(root)
    s.theme_use("clam")
    s.configure("Vertical.TScrollbar",
                background=BG_CARD, troughcolor=BG_PAGE,
                arrowcolor=MUTED,   bordercolor=BORDER)

    Dashboard(root)
    root.mainloop()
