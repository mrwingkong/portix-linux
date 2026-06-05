## Edit the "rc.xml" file under .config/labwc
```
<labwc_config>
  <core>
    <decoration>server</decoration>
    <gap>0</gap>
  </core>
  <theme>
    <name>Mac-Classic-Platinum</name>
    <titlebar>
      <layout>close:min,max,iconify</layout>
    </titlebar>
    <cornerRadius>0</cornerRadius>
  </theme>
  <keyboard>
    <keybind key="W-Return">
      <action name="Execute" command="lxqt-qdbus run qterminal" />
    </keybind>
    <keybind key="W-p">
      <action name="Execute" command="pcmanfm-qt" />
    </keybind>
    <keybind key="A-F2">
      <action name="Execute" command="lxqt-runner" />
    </keybind>
    <keybind key="Print">
      <action name="Execute" command="lxqt-qdbus run screengrab" />
    </keybind>
    <keybind key="W-l">
      <action name="Execute" command="lxqt-leave --lockscreen" />
    </keybind>
    <keybind key="F12">
      <action name="Execute" command="qterminal -d" />
    </keybind>
    <keybind key="W-D">
      <action name="Execute" command="lxqt-qdbus showdesktop" />
    </keybind>
    <keybind key="A-F1">
      <action name="Execute" command="lxqt-qdbus openmenu" />
    </keybind>
    <keybind key="A-F4">
      <action name="Close" />
    </keybind>
    <keybind key="W-a">
      <action name="ToggleMaximize" />
    </keybind>
    <keybind key="W-Left">
      <action name="SnapToEdge" direction="left" combine="yes" />
    </keybind>
    <keybind key="W-Right">
      <action name="SnapToEdge" direction="right" combine="yes" />
    </keybind>
    <keybind key="W-Up">
      <action name="SnapToEdge" direction="up" combine="yes" />
    </keybind>
    <keybind key="W-Down">
      <action name="SnapToEdge" direction="down" combine="yes" />
    </keybind>
    <keybind key="XF86_AudioLowerVolume">
      <action name="Execute" command="lxqt-qdbus volume down" />
    </keybind>
    <keybind key="XF86_AudioRaiseVolume">
      <action name="Execute" command="lxqt-qdbus volume up" />
    </keybind>
    <keybind key="XF86_AudioMute">
      <action name="Execute" command="lxqt-qdbus volume mute" />
    </keybind>
    <keybind key="XF86_MonBrightnessUp">
      <action name="Execute" command="lxqt-config-brightness -i" />
    </keybind>
    <keybind key="XF86_MonBrightnessDown">
      <action name="Execute" command="lxqt-config-brightness -d" />
    </keybind>
  </keyboard>
  <mouse>
    <default />
  </mouse>
</labwc_config>
```
## 2. Download extracted Platinum theme and place in .local/share/themes/Mac-Classic-Platinum/openbox-3 Edit 'themerc'
```
# Window geometry
border.width: 1
window.client.padding.width: 0
window.client.padding.height: 0
window.handle.width: 0
padding.width: 8
padding.height: 1

# Menu geometry
menu.border.width: 1
menu.overlap.x: -3
menu.overlap.y: 0

# Border colors
window.active.border.color: #3a3a3a
window.inactive.border.color: #F0F0F0
menu.border.color: #353535
menu.separator.color: #181818
menu.separator.padding.width: 2
menu.separator.padding.height: 2
window.active.title.separator.color: #181818
window.inactive.title.separator.color: #181818

# Text shadows
window.active.label.text.font: shadow=y
window.inactive.label.text.font: shadow=y
menu.items.font: shadow=y
menu.title.text.font: shadow=y

# Window title justification
window.label.text.justify: center

# =====================
# Active window - stronger polish
# =====================
window.active.title.bg: gradient vertical #f0f0f0 #b8b8b8
window.active.label.bg: flat interlaced
window.active.label.bg.color: #d8d8d8
window.active.label.bg.interlace.color: #e8e8e8
window.active.label.text.color: #111111

window.active.button.unpressed.bg: flat gradient bevell
window.active.button.unpressed.bg.color: #9A9A9A
window.active.button.unpressed.bg.colorTo: #FFFFFF
window.active.button.unpressed.image.color: #181818

window.active.button.pressed.bg: flat gradient bevell
window.active.button.pressed.bg.color: #9A9A9A
window.active.button.pressed.bg.colorTo: #FFFFFF
window.active.button.pressed.image.color: #181818

window.active.button.disabled.bg: flat gradient bevell
window.active.button.disabled.bg.color: #9A9A9A
window.active.button.disabled.bg.colorTo: #FFFFFF
window.active.button.disabled.image.color: #181818

window.active.button.hover.bg: flat gradient bevell
window.active.button.hover.bg.color: #999999
window.active.button.hover.bg.colorTo: #ffffff
window.active.button.hover.image.color: white

window.active.button.toggled.bg: flat gradient bevell
window.active.button.toggled.bg.color: #9A9A9A
window.active.button.toggled.bg.colorTo: #FFFFFF
window.active.button.toggled.image.color: #181818

# =====================
# Inactive windows
# =====================
window.inactive.title.bg: flat
window.inactive.title.bg.color: #DDDDDD
window.inactive.title.bg.colorTo: #141414
window.inactive.label.bg: parentrelative
window.inactive.label.bg.interlace.color: #DDDDDD
window.inactive.label.text.color: black

window.inactive.button.unpressed.bg: flat gradient bevell
window.inactive.button.unpressed.bg.color: #9A9A9A
window.inactive.button.unpressed.bg.colorTo: #FFFFFF
window.inactive.button.unpressed.image.color: #181818

window.inactive.button.pressed.bg: flat gradient bevell
window.inactive.button.pressed.bg.color: #9A9A9A
window.inactive.button.pressed.bg.colorTo: #FFFFFF
window.inactive.button.pressed.image.color: #181818

window.inactive.button.disabled.bg: flat gradient bevell
window.inactive.button.disabled.bg.color: #999999
window.inactive.button.disabled.bg.colorTo: #ffffff
window.inactive.button.disabled.image.color: #181818

window.inactive.button.hover.bg: flat gradient bevell
window.inactive.button.hover.bg.color: #999999
window.inactive.button.hover.bg.colorTo: #ffffff
window.inactive.button.hover.image.color: white

window.inactive.button.toggled.bg: flat gradient bevell
window.inactive.button.toggled.bg.color: #999999
window.inactive.button.toggled.bg.colorTo: #ffffff
window.inactive.button.toggled.image.color: #181818

# Menus
menu.title.bg: raised bevel interlaced
menu.title.bg.color: #CCCCCC
menu.title.bg.interlace.color: #DDDDDD
menu.title.bg.colorTo: #060606
menu.title.text.color: black
menu.title.text.justify: Center
menu.separator.color: #454545
menu.items.bg: flat solid
menu.items.bg.color: #DDDDDD
menu.items.text.color: black
menu.items.justify: center
menu.items.disabled.text.color: #454545
menu.items.active.bg: flat
menu.items.active.bg.color: #202588
menu.items.active.bg.colorTo: #111111
menu.items.active.bg.border.color: #1f1f1f
menu.items.active.justify: center
menu.items.active.text.color: white
menu.items.active.disabled.text.color: #454545

# OSD
osd.border.width: 1
osd.border.color: #000000
osd.bg: flat solid border
osd.bg.color: #181818
osd.bg.border.color: #181818
osd.label.bg: parentrelative
osd.label.text.color: #828282
osd.hilight.bg: flat solid
osd.hilight.bg.color: #000000
osd.unhilight.bg: flat solid
osd.unhilight.bg.color: #353535
