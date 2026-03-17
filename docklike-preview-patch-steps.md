# xfce4-docklike-plugin: Preview Scaling Patch — Step by Step

## Prerequisites

Make sure you have the source cloned and a build directory set up:

```bash
cd ~/xfce4-docklike-plugin
```

---

## Step 1 — `src/GroupMenu.cpp`

Open `src/GroupMenu.cpp` and find:

```cpp
gtk_box_pack_end(GTK_BOX(mBox), GTK_WIDGET(menuItem->mItem), false, true, 0);
```

Replace with:

```cpp
gtk_box_pack_end(GTK_BOX(mBox), GTK_WIDGET(menuItem->mItem), false, false, 0);
```

> Changed `true` to `false` in the `fill` argument so items don't stretch to fill the full popup width.

---

## Step 2 — `src/GroupMenuItem.cpp` — Label setup

Open `src/GroupMenuItem.cpp` and find:

```cpp
gtk_label_set_width_chars(mLabel, 26);
gtk_widget_set_hexpand(GTK_WIDGET(mLabel), true);
```

Replace with:

```cpp
gtk_label_set_max_width_chars(mLabel, 26);
gtk_widget_set_hexpand(GTK_WIDGET(mLabel), true);
gtk_widget_set_halign(GTK_WIDGET(mLabel), GTK_ALIGN_START);
```

> Changed `width_chars` to `max_width_chars` so the label no longer forces a minimum width on the wrapper. Added `GTK_ALIGN_START` to keep the label left-aligned.

---

## Step 3 — `src/GroupMenuItem.cpp` — Remove thumbnail variable

In the same file, find:

```cpp
GdkWindow* window;
GdkPixbuf* pixbuf;
GdkPixbuf* thumbnail;
```

Replace with:

```cpp
GdkWindow* window;
GdkPixbuf* pixbuf;
```

> Removed the unused `thumbnail` variable — the new approach doesn't need it.

---

## Step 4 — `src/GroupMenuItem.cpp` — Replace scaling block

In the same file, find this entire block:

```cpp
/* calculate the new dimensions */
gdouble wRatio = (gdouble)pixbufWidth / (gdouble)previewWidth;
gdouble hRatio = (gdouble)pixbufHeight / (gdouble)previewHeight;

if (hRatio > wRatio)
{
    pixbufWidth = MAX(1, pixbufWidth / hRatio);
    pixbufHeight = MIN(pixbufHeight, previewHeight);
}
else
{
    pixbufWidth = MIN(pixbufWidth, previewWidth);
    pixbufHeight = MAX(1, pixbufHeight / wRatio);
}

thumbnail = gdk_pixbuf_scale_simple(pixbuf, pixbufWidth, pixbufHeight, GDK_INTERP_BILINEAR);

GdkPixbuf* sized = gdk_pixbuf_new(GDK_COLORSPACE_RGB, true, 8, previewWidth, previewHeight);
gint thumbWidth = gdk_pixbuf_get_width(thumbnail);
gint thumbHeight = gdk_pixbuf_get_height(thumbnail);
gint xOffset = (previewWidth - thumbWidth) / 2;
gint yOffset = (previewHeight - thumbHeight) / 2;
gdk_pixbuf_composite(thumbnail, sized, xOffset, yOffset, thumbWidth, thumbHeight, xOffset, yOffset, 1, 1, GDK_INTERP_BILINEAR, 255);

cairo_surface_t* surface = gdk_cairo_surface_create_from_pixbuf(sized, scale_factor, nullptr);

gtk_image_set_from_surface(mPreview, surface);

cairo_surface_destroy(surface);
g_object_unref(sized);
g_object_unref(thumbnail);
g_object_unref(pixbuf);
```

Replace with:

```cpp
/* fit within previewWidth x previewHeight, no crop, height adapts for wide windows */
gdouble scale = MIN((gdouble)previewHeight / (gdouble)pixbufHeight,
                    (gdouble)previewWidth  / (gdouble)pixbufWidth);

gint canvasWidth  = MAX(1, (gint)(pixbufWidth  * scale));
gint canvasHeight = MAX(1, (gint)(pixbufHeight * scale));

GdkPixbuf* sized = gdk_pixbuf_new(GDK_COLORSPACE_RGB, true, 8, canvasWidth, canvasHeight);
gdk_pixbuf_fill(sized, 0x00000000);
gdk_pixbuf_composite(pixbuf, sized, 0, 0, canvasWidth, canvasHeight, 0, 0, scale, scale, GDK_INTERP_BILINEAR, 255);

cairo_surface_t* surface = gdk_cairo_surface_create_from_pixbuf(sized, scale_factor, nullptr);

gtk_image_set_from_surface(mPreview, surface);
gtk_widget_set_size_request(GTK_WIDGET(mPreview), canvasWidth / scale_factor, canvasHeight / scale_factor);
gtk_widget_set_size_request(GTK_WIDGET(mGrid), canvasWidth / scale_factor, Settings::previewHeight);
gtk_widget_set_valign(GTK_WIDGET(mPreview), GTK_ALIGN_CENTER);
gtk_widget_set_vexpand(GTK_WIDGET(mPreview), true);
gtk_widget_set_size_request(GTK_WIDGET(mItem), canvasWidth / scale_factor, -1);

cairo_surface_destroy(surface);
g_object_unref(sized);
g_object_unref(pixbuf);
```

> This is the core change. The new approach scales each window to fit within `previewWidth × previewHeight` without cropping or bars. Narrow windows scale to full height. Wide windows shrink in height so the width fits. The wrapper resizes dynamically to match the content.

---

## Step 5 — Build and Install

```bash
cd ~/xfce4-docklike-plugin/build
ninja -j$(nproc)
sudo cp src/libdocklike.so /usr/lib/x86_64-linux-gnu/xfce4/panel/plugins/
xfce4-panel -r
```

---

## Optional — CSS Customization

No rebuild needed. Create this file to customize spacing and appearance:

```bash
mkdir -p ~/.config/xfce4-docklike-plugin
nano ~/.config/xfce4-docklike-plugin/gtk.css
```

Paste this content:

```css
.xfce-docklike-window { border: none; background-color: transparent; }
.xfce-docklike-window .menu { border-radius: 6px; box-shadow: 0px 4px 12px rgba(0, 0, 0, 0.4); background-color: alpha(@menu_bgcolor, 0.95); }
.xfce-docklike-window .menu_item grid { margin: 0px; padding: 4px; }
.xfce-docklike-window .menu_item .preview { margin: 2px 0px; }
```

Then restart the panel:

```bash
xfce4-panel -r
```
