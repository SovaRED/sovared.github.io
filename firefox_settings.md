# Налаштування для кращої роботи Firefox

в адресному рядку ввести
``` 
abaut:config
```
та прийняти все

## _Вминути WebRender_
Змінити на `true`
``` 
gfx.webrender.all
widget.wayland-dmabuf-vaapi
widget.wayland-dmabuf-vaapi.enabled
``` 
## _Вмикнути кінетичний скролл_
Змінити на `true`
``` 
apz.gtk.kinetic_scroll.enabled
mousewheel.default.delta_multiplier_
``` 
Налаштувати швидкість
``` 
mousewheel.default.delta_multiplier_x 350
mousewheel.default.delta_multiplier_y 350
mousewheel.default.delta_multiplier_z 350
``` 
