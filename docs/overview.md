# Project Alice UI Template Editor

## Introduction

The ui template editor is responsible for managing the common ui template file (`the.tui`) that individual ui project files (`*.aui`) can use to define the appearance of controls and windows. Doing so means that (a) there is no need to create individual textures whenever a control of new dimensions is added and (b) the created ui will have a common look and feel that can be controlled as a whole by editing the ui template file.

## SVG usage

Icons are defined using standard svg files. When rendered, an icon will be resized to fit the destination area, which will result in stretching if the icon's `viewBox` does not share the same relative proportions. Some limitations on svg rendering may be encountered because of limitations in the relatively simple svg renderer used. If it does not render as desired in the template editor, it won't render properly in-game either.

When used, the renderer will attempt to match the color of the icon to the defined color for the control, if available. This is done, for example, to allow the icon for a disabled button to take on the disabled color if desired. To enable this, the svg must mark all elements that should have their color changed with `class="primarycolor"`. Any marked elements will have their stroke and fill color changed to match the target color for the icon. (You can see a preview of this in the template editor as it will produce a sample red render of the icon so you can see what exactly is changed.) Make sure that you add `fill-opacity="0"` and/or `stroke-opacity="0"` to elements that you don't want the stroke or fill to render for when the new color is applied. Finally, due to limitations in the svg renderer, the new color cannot be applied to elements that have their style defined in a single `style="..."` statement (I don't know why this is the case either; it just doesn't work). Thus, such elements must have their `fill="#000000"`, etc properties defined individually.

## ASVG usage

.asvg files (affine svg) define the variable-sized background regions that are used to render controls and windows. An asvg file is the same as an svg file except that chosen numerical parameters can be controlled by affine transformations, which allows for things like a rounded rect that has corners of a fixed size even when it is rendered at different proportions and scales.

All asvg rendering is done in terms of the grid size defined for the .aui project file (which is an additional reason to set that size to a sensible value). One grid unit corresponds to 500 units within the internal coordinate system of the asvg file. So, if you defined a line to start at 500,500 within the asvg file and the grid size was 8, that line would start at 8,8 pixels within the final render of that background.

Numerical replacements inside an asvg file are created by insertion markers in the format `[[Z;XXX;YYY]]` where `Z` is a single letter denoting the *formula*, `XXX` is a numerical value representing the *scale* and `YYY` is a numerical value representing the *offset*. If there is a quotation mark either before, after, or on both sides of an insertion marker, the resulting value when the file is processed for rendering will be surrounded by quotation marks. Thus if you write `width = "[[W;1000;0]]"` the renderer may see `width = "100"`. Because of this, it is important to remember to add spaces around insertion markers that you don't want to accidentally end up in quotation marks, such as `viewBox = "0 0 [[W;1000;0]] [[H;1000;0]] " `.

The formula for how an insertion marker is converted into a numerical value depends partly on the base size given for the asvg file in the `.tui` file. The asvg files I have written all use 1000,1000 as their base size, but that is not necessary. To calculate the value that an insertion marker is converted to, first we find the horizontal and vertical *base value*. The *horizontal base value* is the width of the render (in grid units) `x` 500 `/` the *base horizontal size* (usually 1000). The *vertical base value* is the height of the render (in grid units) `x` 500 `/` the *base vertical size* (usually 1000). We then consult the table below based on the *formula letter* of the insertion marker.

| letter | result |
|:-------------:|-------------|
| W | the final value is the *horizontal base value* `x` *scale* `+` *offset* |
| H | the final value is the *vertical base value* `x` *scale* `+` *offset* |
| S | the final value is the smaller of the horizontal and vertical *base values* `x` *scale* `+` *offset*  |
| L | the final value is the larger of the horizontal and vertical *base values* `x` *scale* `+` *offset* |
| D | the final value is *sqrt*(*horizontal base value* squared `+` *vertical base value* squared) `x` *scale* `+` *offset* |
| P | the final value is the value that will result in a length of *scale* pixels in size when rendered`+` *offset* |

### Example usage: a path command

Concretely, let's walk through how these substitutions can be used in path command within an asvg of base size 1000,1000 (you may also wish to consult the svg documentation if you are unfamiliar with the syntax of the path command)

```
d = "M [[W;1000;-500]], [[H;0;500]]
m [[P;1.25;0]], [[P;-1.25;0]]
h [[W;-1000;1000]]
h [[P;-2.75;0]]
v [[H;1000;-1000]]
v [[P;2.75;0]] "
```

- `M [[W;1000;-500]], [[H;0;500]]` -- The pen for the path is moved to an (absolute) position. `[[W;1000;-500]]` resolves to 500 units to the left of the right edge, i.e. one grid unit from the right. `[[H;0;500]]` resolves to 500 units from the top, i.e. one grid unit from the top.
- `m [[P;1.25;0]], [[P;-1.25;0]]` -- The pen for the path is moved relatively by an amount. In this case it will be moved `[[P;1.25;0]]`, i.e. 1.25 pixels rightwards in the final render and [[P;-1.25;0]], i.e. 1.25 pixels up in the final render. (This is done because the pen has a width of 2.5 pixels in this path and it is being moved so that the entire line falls outside of the grid unit.)
- `h [[W;-1000;1000]]` -- Draw a horizontal line for `[[W;-1000;1000]]` units, which is -1000 `x` *horizontal scale* i.e. the entire width of the region to the left plus 1000 (so shorter by that amount, since the amount is negative). So the line will be the width of the entire region minus two grid units in length.
- `h [[P;-2.75;0]]` -- Extend the line by an additional 2.75 pixels to the left.
- `v [[H;1000;-1000]]` -- Draw a vertical line for `[[H;1000;-1000]]`. This is the same as the command above, except that `v` means that the direction of the line is vertical, and by using the `H` function letter instead of `W` the length is proportional to the height of the render. The line will extend downwards for the height of the render minus two grid units.
- `v [[P;2.75;0]] "` -- Extend the line by an additional 2.75 pixels downwards. Note the space before the `"` to avoid having the inserted replacement being surrounded by quotations, which would not be a valid svg file.

## Brief note on windows and layout regions

A layout region template can be used to control the appearance of the left and right buttons for paged layouts (when present) and to generate a background that will cover the entire layout region (note: this includes the space for the margins defined for the layout; the intention is to use the marginal space to ensure that any border defined in the background will not be covered by controls). Each window template should define a default layout region template. When a window is given a template inside the UI editor, this default layout region template will be applied to all layout regions within that window unless they are given a specific layout region template of their own.

## Alternate templates

In the UI editor, one of the properties for a window, "Has an alternate template set," allows you to optionally define alternate templates for the window itself and any of its controls (but not currently for layout regions). This alternate set is designed for use by windows that serve as items in a generated list. In such a list, every other item will have its alternate set picked for rendering, allowing you to use backgrounds and other design choices to distinguish adjacent items. The alternate set can also be changed manually via the generated `set_alternate` function for the window if you need it for some other reason.