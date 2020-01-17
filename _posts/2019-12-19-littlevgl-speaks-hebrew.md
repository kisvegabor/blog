---
layout: post
title: "LittlevGL speaks Hebrew"
author: "amirgon"
cover: /assets/bidi/aleph.png
image:
  path: /assets/bidi/cover.jpg
  height: 224
  width: 884

---


![LittlevGL speaks Hebrew](/assets/bidi/cover.jpg)

[Release 6.1](https://blog.littlevgl.com/2019-12-06/release_v6_1) of LittlevGL contains exciting new features, one of them is **Bidirectional support**.

More information about this is available [on the docs](https://docs.littlevgl.com/en/html/overview/font.html#bidirectional-support).
> Most of the languages use Left-to-Right (LTR for short) writing direction, however some languages (such as Hebrew) uses Right-to-Left (RTL for short) direction.
> LittlevGL not only supports RTL texts but supports mixed (a.k.a. bidirectional, BiDi) text rendering too.

Supporting mixed bidirectional scripts on LittlevGL required following the convoluted [Unicode Bidirectional Algorithm](https://unicode.org/reports/tr9/) specifications and creating a custom implementation for resource constrained devices.  
While I participated in this effort, most of the work was actually done by @kisvegabor, which is very impressive since he cannot read any RTL script!  

RTL scripts are used by more than 400 million people around the world. The most used script is Arabic script.  
My personal interest is displaying Hebrew on embedded devices, so this is what I'm going to talk about, but Bidirectional support is beneficial for Arabic scripts as well.

# Displaying Hebrew on Embedded devices

The steps for displaying Hebrew on LittlevGL are simple and straightforward:

## 1. Generate a Font

LittlevGL uses its own font format. You can use [LittlevGL font converter](https://github.com/littlevgl/lv_font_conv) to convert a standard `ttf` font to LittlevGL font.
There is also an [online font converter](https://littlevgl.com/ttf-font-to-c-array), but I chose to use the command line tool.  
The font converter installation and usage instructions are [well documented](https://github.com/littlevgl/lv_font_conv#install-the-script), and this is the command line I used:
```
node lv_font_conv.js --font FrankRuehlCLM-Medium.ttf -r 0x20-0x7F -r 0x5d0-0x5ea --size 16 --format lvgl --bpp 4 --no-compress -o lv_font_heb_16.c
```

As input I provided a font from the [Culmus](http://culmus.sourceforge.net/) project, but you could use any other `ttf` font.
Unicode ranges `0x20-0x7F` and `0x5d0-0x5ea` cover only the most basic Latin and Hebrew characters.

This command generates a new C file `lv_font_heb_16.c`, which represnts the character glyphs and can be used with LittlevGL.

## 2. Build the font

The font C file needs to be linked statically with the rest of your project, because unfortunately, it's not possible to load a font file at runtime yet. This is [being considered](https://github.com/littlevgl/lvgl/issues/1237), however.  
So either put it under `lvgl/src/lv_font/` or specify its location on your Makefile, depending on how you build lvgl.

Then edit `lv_conf.h`, to associate it with lvgl:
```c
#define LV_FONT_CUSTOM_DECLARE  LV_FONT_DECLARE(lv_font_heb_16)
#define LV_FONT_DEFAULT         &lv_font_heb_16
```

Actually, you don't have to set `LV_FONT_DEFAULT`. But if you don't, you'd have to explicitly set the hebrew font on every style of every object you want to display (at least on lvgl v6).

## 3. Enable Bidi
Bidirectional support is not enabled by default.
To enable it, edit `lv_conf.h`:
```c
#define LV_USE_BIDI     1
```

## 4. Display Hebrew!

LittlevGL supports `UTF8` so you can just set utf8 text for any label or text box.

# Setting the Base Dir

When displaying RTL text, there is significance to the base directionality of the object that contains the text.  
The base dir affects the alignment of the text inside a label, the way intermixed RTL and LTR texts are displayed together and even the way the object itself is displayed and aligned.  
For example, a table with LTR base dir would have its first column as the leftmost colunm, while an RTL table first column would be the **rightmost** column.  
If an entire page or screen is supposed to be RTL, it's enough to either set `LV_BIDI_BASE_DIR_DEF` on `lv_conf.h`, or alternatively set the screen (or page) base dir on runtime.

# How does it look like?

A super simple eaxmple in C:
```c
    /*Create a style which uses a font with Hebrew characters too*/
    LV_FONT_DECLARE(lv_font_heb_16);
    static lv_style_t style;
    lv_style_copy(&style, &lv_style_plain);
    style.text.font = &lv_font_heb_16;

    /*Create a label and set its text*/
    lv_obj_t * label = lv_label_create(lv_scr_act(), NULL);
    lv_label_set_style(label, LV_LABEL_STYLE_MAIN, &style);
    lv_label_set_text(label, "שלום LittlevGL");
    lv_obj_align(label, NULL, LV_ALIGN_CENTER, 0, 0);
```

![Hebrew text in LittlevGL](/assets/bidi/simple.png)

Following are a few more examples in [Micropython](https://docs.littlevgl.com/en/html/get-started/micropython.html), but the same thing can also be achieved with C.

Instead of changing the base dir in `lv_conf.h`, I'm setting it for my screen. The base dir for all other objects created on that screen will also be set to RTL.
```python
scr = lv.obj()
scr.set_base_dir(lv.BIDI_DIR.RTL)
```

Create 3 tabs
```python
tabview = lv.tabview(scr)
tab1 = tabview.add_tab("1. אחד")
tab2 = tabview.add_tab("2. שתיים")
tab3 = tabview.add_tab("3. שלוש")
```

Create a table
```python
style_cell1 = lv.style_t()
lv.style_copy(style_cell1, lv.style_plain)
style_cell1.body.border.width = 1
style_cell1.body.border.color = lv.color_hex3(0xFFF)

style_cell2 = lv.style_t()
lv.style_copy(style_cell2, lv.style_plain)
style_cell2.body.border.width = 1
style_cell2.body.border.color = lv.color_hex3(0xFFF)
style_cell2.body.main_color = lv.color_hex3(0xAAA)
style_cell2.body.grad_color = lv.color_hex3(0xAAA)

table = lv.table(tab1)
table.set_style(table.STYLE.CELL1, style_cell1);
table.set_style(table.STYLE.CELL2, style_cell2);
table.set_style(table.STYLE.BG, lv.style_transp_tight);
table.set_col_cnt(2);
table.set_row_cnt(4);

table.set_cell_align(0, 0, lv.label.ALIGN.CENTER);
table.set_cell_align(0, 1, lv.label.ALIGN.CENTER);

table.set_cell_type(0, 0, 2);
table.set_cell_type(0, 1, 2);

table.set_cell_value(0, 0, "שם");
table.set_cell_value(1, 0, "תפוח");
table.set_cell_value(2, 0, "בנננה");
table.set_cell_value(3, 0, "לימון");

table.set_cell_value(0, 1, "מחיר");
table.set_cell_value(1, 1, "$7");
table.set_cell_value(2, 1, "$4");
table.set_cell_value(3, 1, "$6");
```

And this is how it looks like:

![Tabs and Table](/assets/bidi/table.png)

You can see how both the tabs and table columns are ordered from right to left, since they inherited RTL directionality from `scr`.  
Here is another example, of intermixed RTL and LTR text:

![Text Box](/assets/bidi/textbox.png)


### You can see this example, run it and even change it using [LittlevGL online simulator](https://docs.littlevgl.com/en/html/get-started/micropython.html#online-simulator) on [this link](https://sim.littlevgl.com/v6/micropython/ports/javascript/lvgl_editor.html?script_startup=https://gist.githubusercontent.com/amirgon/7bf15a66ba6d959bbf90d10f3da571be/raw/8684b5fa55318c184b1310663b187aaab5c65be6/init_lv_mp_js.py&script=https://gist.githubusercontent.com/amirgon/8526af6d09b4f07f784410c959647a33/raw/f29857159d0395efe2d11d2ecfccd18e015738f6/lvgl_heb_test1.py).

# What about Arabic?

Arabic script is an RTL script as well, but we found out it was more complex to render than Hebrew.
For LittlevGL to support arabic, it would need a few extra features that are still missing today:

- Ligatures
- Diacritics
- Context-sensitive text shaping (each character can have multiple shapes)

If you are willing to help out implementing these missing features in LittlevGL, please contact us on [the forum](https://forum.littlevgl.com/)!

