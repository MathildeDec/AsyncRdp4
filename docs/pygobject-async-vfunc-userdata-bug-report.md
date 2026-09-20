# Rapport de bug PyGObject — prêt à déposer

Ce document a été rédigé le 2026-09-11 (voir `docs/sessions/session-25.md`)
après recherche : aucun rapport existant trouvé sur
`gitlab.gnome.org/GNOME/pygobject/-/issues` correspondant exactement à ce
scénario (recherche par mots-clés uniquement — l'index de recherche
GitLab lui-même n'a pas pu être interrogé directement, cf.
`docs/sessions/session-25.md` pour le détail). Contenu ci-dessous en
anglais, prêt à coller tel quel dans un nouveau ticket sur
<https://gitlab.gnome.org/GNOME/pygobject/-/issues/new> — c'est ce
projet-ci (`asyncrdp`) qui est concerné, mais le bug lui-même est dans
PyGObject, pas dans ce dépôt.

---

## Title

`user_data` is lost (arrives as `None`) when a Python-implemented async
vfunc with a `(GAsyncReadyCallback callback, gpointer user_data)` pair is
invoked from C — reproduced with `GdkContentProvider.write_mime_type_async`

## Environment

- PyGObject 3.48.2
- GTK 4.14.5 (Ubuntu package `4.14.5+ds-0ubuntu0.9`/`.10`)
- Python 3.12.3
- Ubuntu 24.04.4 LTS, X11 session (also reproduced under Xvfb)

## Summary

When GDK calls a Python-overridden `write_mime_type_async` vfunc on a
custom `Gdk.ContentProvider` subclass, the `user_data` parameter received
by the Python method is always `None`, even though the real C-side
`gpointer` passed by `gdk_clipboard_write_async()` is a genuine, valid
`GTask*` at that point (confirmed by reading GTK's own source, see
below). Because `user_data` cannot be recovered, any implementation of
`write_mime_type_async` that later needs to hand a result back through
the received `callback`/`user_data` pair (the normal GAsyncReadyCallback
pattern) cannot do so correctly, and the caller-side completion crashes
with:

```
GLib-GIO-CRITICAL **: g_task_return_boolean: assertion 'G_IS_TASK (task)' failed
GLib-GObject-CRITICAL **: g_object_unref: assertion 'G_IS_OBJECT (object)' failed
```

This makes it impossible to implement `Gdk.ContentProvider` (specifically
`write_mime_type_async`/`write_mime_type_finish`) for image formats in
pure Python — the standard workaround for the well-known "GTK4 does not
serialize `GdkTexture` to `image/png` etc. out of the box in some
configurations" limitation (the technique used by Vim's `gui_gtk4_cb.c`
in C) does not work in PyGObject.

## Steps to reproduce

```python
import gi
gi.require_version("Gtk", "4.0")
from gi.repository import Gtk, Gdk, Gio, GLib, GdkPixbuf


class MinimalImageProvider(Gdk.ContentProvider):
    def __init__(self, png_bytes):
        super().__init__()
        self._png_bytes = png_bytes

    def do_ref_formats(self):
        return Gdk.ContentFormats.new(["image/png"])

    def do_get_value(self, gvalue):
        gvalue.set_object(self)
        return True

    def do_write_mime_type_async(self, mime_type, stream, io_priority,
                                  cancellable, callback, user_data):
        print("user_data received from GDK:", repr(user_data))  # -> None
        task = Gio.Task.new(self, cancellable, callback, user_data)
        try:
            stream.write_bytes(self._png_bytes, cancellable)
            task.return_boolean(True)
        except GLib.Error as exc:
            task.return_error(exc)

    def do_write_mime_type_finish(self, result):
        return result.propagate_boolean()  # executes fully, returns True


def on_activate(app):
    win = Gtk.ApplicationWindow(application=app)
    win.present()

    pixbuf = GdkPixbuf.Pixbuf.new(GdkPixbuf.Colorspace.RGB, True, 8, 4, 2)
    pixbuf.fill(0xFF0000FF)
    texture = Gdk.Texture.new_for_pixbuf(pixbuf)

    provider = MinimalImageProvider(texture.save_to_png_bytes())
    Gdk.Display.get_default().get_clipboard().set_content(provider)

    # Then, from another process/terminal on the same X11 display:
    #   xclip -o -selection clipboard -t image/png > /tmp/out.png
    # triggers the crash below and delivers 0 bytes.


app = Gtk.Application(application_id="org.example.pygobject.repro")
app.connect("activate", on_activate)
app.run(None)
```

## Expected result

`user_data` should be the real value GDK passed internally (or, at
minimum, `write_mime_type_finish`'s eventual completion via `callback`
should not crash), and `xclip -o ... -t image/png` should receive the
PNG bytes written to `stream`.

## Actual result

- `user_data` printed from Python is always `None`.
- `do_write_mime_type_finish` runs to completion and returns `True`
  (confirmed with debug prints — this part of the round-trip works).
- Immediately after, the following appears on stderr and the external
  reader receives 0 bytes:
  ```
  GLib-GIO-CRITICAL **: g_task_return_boolean: assertion 'G_IS_TASK (task)' failed
  GLib-GObject-CRITICAL **: g_object_unref: assertion 'G_IS_OBJECT (object)' failed
  ```

## Root-cause analysis (source reading + empirical testing)

Reading GTK's own source (tag `4.14.5`, matching the installed package,
via <https://github.com/GNOME/gtk> — a read-only mirror of
`gitlab.gnome.org/GNOME/gtk`):

- `gdk/gdkclipboard.c::gdk_clipboard_write_async()` creates a real,
  valid `GTask *task = g_task_new(clipboard, cancellable, callback,
  user_data)`, then — when the requested mime type is one the provider
  already advertises — calls
  `gdk_content_provider_write_mime_type_async(priv->content, mime_type,
  stream, io_priority, cancellable, gdk_clipboard_write_done, task)`.
  **`task` is a genuine, valid C pointer at this exact point.**
- `gdk_clipboard_write_done()` (the callback above) calls
  `gdk_content_provider_write_mime_type_finish(content, result, &error)`
  and then, on success, `g_task_return_boolean(task, TRUE)` — this is
  the exact line whose assertion fails in practice, which matches: the
  `finish()` call succeeds first (confirmed above), then the very next
  statement crashes.
- Nothing in `gdk/x11/gdkclipboard-x11.c` or
  `gdk/x11/gdkselectionoutputstream-x11.c` (the X11-specific path that
  leads here from a `SelectionRequest` X11 event) wraps `task` in any
  additional object — the C logic, read in full, is sound.

This means the C-side `task`/`user_data` genuinely exists and is valid
when GDK calls into the Python vfunc override — the loss happens in
PyGObject's marshaling of this specific vfunc call from C into Python.

**Ruled out empirically** (i.e. the crash is identical regardless):
- Using `Gio.SimpleAsyncResult` instead of `Gio.Task` in
  `do_write_mime_type_async`/`_finish`.
- Passing `source_object=None` instead of `self` to the inner task
  constructor.
- Keeping an explicit strong Python reference to the provider (rules
  out premature garbage collection as the cause).
- Creating a **second, fully independent** `Gio.Task` that never
  receives the `(callback, user_data)` pair at all, and instead calling
  the received `callback` **directly and manually** — tried with both
  `callback(self, inner_task, user_data)` (3 args) and
  `callback(self, inner_task)` (2 args, in case `user_data` is meant to
  be carried internally by the `callback` wrapper object itself). Both
  produce the exact same crash, and `user_data` is confirmed `None` in
  both cases *before* `callback` is even invoked — i.e. the value is
  already unusable at the point `do_write_mime_type_async` receives it,
  regardless of what the Python code subsequently does with it.

This last point is the key finding: no implementation strategy *inside*
`do_write_mime_type_async`/`do_write_mime_type_finish` can work around
this, because the information needed (the real `user_data`) is already
gone by the time Python code runs. A fix would need to live in
PyGObject's vfunc-call marshaling for this parameter (the `gpointer
user_data` half of a `(GAsyncReadyCallback, gpointer)` pair arriving as
*incoming* vfunc arguments — as opposed to being supplied by Python code
calling *out* to a C async function, which works correctly and is well
covered by existing tests/overrides).

## Suspected mechanism

`write_mime_type_async` is introspected with a closure/scope
relationship between its `callback` and `user_data` parameters
(`GI_SCOPE_TYPE_ASYNC` in gobject-introspection terms — the same scope
type used for ordinary outbound `_async()` calls, where PyGObject is
well-tested at *constructing* such a pair to hand to C). The direction
exercised here is the reverse: PyGObject receiving such a pair as
*incoming* vfunc parameters when C calls into a Python override. This
is a much less common path (most Python-overridden vfuncs — `do_get_value`,
`do_ref_formats`, ordinary signal handlers — carry no callback/user_data
closure at all), so it may simply be undertested. This is offered as a
starting hypothesis for maintainers, not a confirmed diagnosis of the
PyGObject-internal code path itself, which was not read as part of this
report (only GTK's side was).

## Workaround

None found that stays in pure Python. Likely requires either a small
compiled C `GdkContentProvider` subclass (exposed to Python via
whatever binding mechanism, e.g. cffi/GObject Introspection typelib for
a tiny custom library), or a PyGObject-side fix.
