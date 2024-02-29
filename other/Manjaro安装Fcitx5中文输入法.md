## 删除Fcitx4
```sudo pacman -Rs $(pacman -Qsq fcitx)```

## 安装Fcitx5
```sudo pacman -S fcitx5 fcitx5-configtool fcitx5-qt fcitx5-gtk fcitx5-chinese-addons fcitx5-material-color kcm-fcitx5 fcitx5-lua```

## 配置ENV  
这里用的 profile  
```sudo vim /etc/profile```
```
export XMODIFIERS=@im=fcitx
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
```

## 使用皮肤
使用刚才安装的fcitx5-material-color  
```vim ~/.config/fcitx5/conf/classicui.conf```
```
# 垂直候选列表
Vertical Candidate List=False

# 按屏幕 DPI 使用
PerScreenDPI=True

# Font (设置成你喜欢的字体)
Font="思源黑体 CN Medium 13"

# 主题
Theme=Material-Color-Pink
```
