##### 键盘映射修改
- 创建.Xmodmap
- 修改映射
```
remove Lock = Caps_Lock
remove Mod1 = Alt_R
remove Mod5 = Scroll_Lock

keycode 30 = u U Home
keycode 31 = i I Up
keycode 32 = o O End
keycode 40 = d D Delete
keycode 41 = f F Prior
keycode 44 = j J Left
keycode 45 = k K Down
keycode 46 = l L Right
keycode 56 = b B BackSpace
keycode 57 = n N Next
keycode 64 = Alt_L
keycode 66 = Mode_switch
keycode 113 = Caps_Lock

add Lock = Caps_Lock
add Mod1 = 0x007D 0x009C Alt_L Alt_L
add Mod4 = 0x007F 0x0080
add Mod5 = Mode_switch ISO_Level3_Shift
```
- 命令生效

`xmodmap .Xmodmap`
