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
