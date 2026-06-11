---
title: Diagrams
---

# An Inkscape to TikZ diagram workflow
This is how I make diagrams for maths documents.
This is only relevant to you if you can edit/compile LaTex on your own computer, i.e. not using OverLeaf.

My setup is:
- Use Inkscape to make and edit `.svg` files.
- Use [svg2tikz](https://github.com/xyz2tex/svg2tikz) to translate the `.svg` files to TikZ.
- A small LaTex macro is used to include these files (one per diagram) in to the main LaTex document.
- A Python watchdog script looks for changes to `.svg` files in the relevant directory and creates/updates the TikZ code files automatically.

With this setup, the workflow is:
1. Run the watchdog script.
1. Create/edit `.svg` files using Inkscape.

Then, all edits are then automatically and instantly reflected in LaTex.
If your LaTex compiler runs each time a file is changes, the document will be re-compiled to reflect the changed diagram.

Here's why I think this is nice.
- This is non-prescriptive.
If you would rather use TikZ, you can.
Just put that TikZ in an appropriately named file.
- You can use a combination of visual and textual editing.
Make a difficult to visualise object in Inkscape, and copy the generated TikZ code in to the relevant TikZ file.

In the maths I do, I need to draw a lot of curved paths (think braid diagrams etc.) and textually editing numbers defining complicated Bézier curves is just not practical.

---

### How to

These instructions are for Linux.

The first step is to get access to a `svg2tikz`.
To do so, you create a Python environment (somewhere where it can stay, like your home directory) and then install the relevant [pip package](https://pypi.org/project/svg2tikz/) in that Python environment.
You should also install `watchdog` in this environment.
Then, you will find a svg2tikz binary in `path-to-python-env/bin/svg2tikz`.

Then, make a Python file containing the following.
This is a watchdog script which does a bit of wrangling on the output of `svg2tikz` towards this use-case.
```python
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
from datetime import datetime
from pathlib import Path
import subprocess
import threading
import time
import sys

SCRIPT_DIR = Path(__file__).resolve().parent
DEBOUNCE_SECONDS = 2.0


class SVGHandler(FileSystemEventHandler):
    def __init__(self, out_dir: Path):
        self._out_dir = out_dir
        self._lock = threading.Lock()
        self._pending: dict[Path, threading.Timer] = {}
        self._in_flight: set[Path] = set()

    def _schedule(self, svg_path: Path):
        with self._lock:
            existing = self._pending.pop(svg_path, None)
            if existing:
                existing.cancel()
            timer = threading.Timer(DEBOUNCE_SECONDS, self._run_conversion, args=[svg_path])
            self._pending[svg_path] = timer
            timer.start()

    def _run_conversion(self, svg_path: Path):
        with self._lock:
            self._pending.pop(svg_path, None)
            if svg_path in self._in_flight:
                # Conversion still running — reschedule for after it finishes
                timer = threading.Timer(DEBOUNCE_SECONDS, self._run_conversion, args=[svg_path])
                self._pending[svg_path] = timer
                timer.start()
                return
            self._in_flight.add(svg_path)

        try:
            _convert(svg_path, self._out_dir)
        finally:
            with self._lock:
                self._in_flight.discard(svg_path)

    def on_modified(self, event):
        if not event.is_directory and event.src_path.endswith(".svg"):
            self._schedule(Path(event.src_path).resolve())

    def on_created(self, event):
        if not event.is_directory and event.src_path.endswith(".svg"):
            self._schedule(Path(event.src_path).resolve())


def _convert(svg_path: Path, out_dir: Path):
    out_path = out_dir / (svg_path.stem + ".tex")
    tmp_figonly = out_dir / (svg_path.stem + "_tmp_figonly.tex")
    tmp_codeonly = out_dir / (svg_path.stem + "_tmp_codeonly.tex")

    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    print(f"[{timestamp}] Converting {svg_path.name} -> {out_path.name}", flush=True)

    svg2tikz = str(SCRIPT_DIR / "venv/bin/svg2tikz")
    base_args = [
        svg2tikz, str(svg_path),
        "-t", "raw",
        "--markings", "interpret",
        "--arrow", "stealth",
    ]

    try:
        subprocess.run(
            base_args + ["--output", str(tmp_figonly), "--codeoutput", "figonly"],
            check=True, capture_output=True,
        )
        subprocess.run(
            base_args + ["--output", str(tmp_codeonly), "--codeoutput", "codeonly"],
            check=True, capture_output=True,
        )

        with open(tmp_figonly) as f:
            color_defs = [line for line in f if line.strip().startswith(r"\definecolor")]
        with open(tmp_codeonly) as f:
            code_lines = f.readlines()

        with open(out_path, "w") as f:
            f.writelines(color_defs + ["\n"] + code_lines)

        print(f"[{datetime.now().strftime('%H:%M:%S')}] Done: {out_path.name}", flush=True)

    except subprocess.CalledProcessError as e:
        stderr = e.stderr.decode(errors="replace").strip() if e.stderr else ""
        print(f"[ERROR] svg2tikz failed for {svg_path.name}: {stderr or e}", flush=True)
    except OSError as e:
        print(f"[ERROR] File I/O error for {svg_path.name}: {e}", flush=True)
    finally:
        for tmp in (tmp_figonly, tmp_codeonly):
            try:
                tmp.unlink(missing_ok=True)
            except OSError:
                pass


if __name__ == "__main__":
    project_dir = Path(sys.argv[1]).resolve() if len(sys.argv) > 1 else Path.cwd()
    svg_dir = project_dir / "svg_src"
    out_dir = project_dir / "figs"

    if not svg_dir.exists():
        print(f"Error: {svg_dir} not found", file=sys.stderr)
        sys.exit(1)

    out_dir.mkdir(exist_ok=True)
    handler = SVGHandler(out_dir)
    observer = Observer()
    observer.schedule(handler, str(svg_dir), recursive=False)
    observer.start()
    print(f"Watching {svg_dir} for .svg changes... (Ctrl-C to stop)", flush=True)
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    observer.join()
```

This expects the Python virtual environment to be alongside wherever this script lives.
Run this script using the python virtual environment in which `svg2tikz` and `watchdog` is installed.

I have this setup using an alias.
```bash
alias svg-watch="/home/sean/.scripts/svg_conversion/venv/bin/python3.13 /home/sean/.scripts/svg_conversion/svg_watchdog.py"
```
So, I run `svg-watch` in my LaTex project directory,
I put `.svg` files in `svg_src/` in that project directory, and converted TikZ files are put in `figs/` in the project directory.

Then, to include the diagram in your LaTex, use the following macro.

```latex
\newcommand{\includetikz}[2][]{
	\begin{tikzpicture}[#1,every node/.append style={scale=1}, inner sep=0pt, outer sep=0pt]
		\input{#2}
	\end{tikzpicture}%
}
```

And add use that in your LaTex document like so.

```latex
\begin{figure}
    \includetikz[]{figs/my_diagram.tex}
\end{figure}
```

You can add optional arguments like base-point etc.

```latex
\begin{figure}
    \includetikz[baseline=(baseline_text.center)]{figs/all_arcs.tex}
\end{figure}
```
---

**Get in touch if you have any questions.**
