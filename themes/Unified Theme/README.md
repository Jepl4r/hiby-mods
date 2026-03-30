# Unified Theme for Hiby R3 Pro II

# List of changes

## Screensaver

Added an arrow at the bottom middle of the page to show the swipe direction by adding a .png file and editing the corresponding .view file `hiby_screensavers.view` 

![ ](https://github.com/Jepl4r/hiby-mods/blob/unified_theme/themes/Unified%20Theme/Screenshots/Screensaver.png)

## Main Menu

Modified the `hiby_launcher_apps.view` file to show a rounded rectangle under each icon to make the page feel more in-line with the rest of the menus.
The same has been done to the icons, they now look similar to the rest.

![ ](https://github.com/Jepl4r/hiby-mods/blob/unified_theme/themes/Unified%20Theme/Screenshots/Main%20Page.png)

## Stream Media page

Modified the icons of the dark theme to more closely match the ones from the light theme.

![ ](https://github.com/Jepl4r/hiby-mods/blob/unified_theme/themes/Unified%20Theme/Screenshots/Stream%20Media.png)

## Now Playing page

Modified the favorites icon to match the one from iOS, it is now a star.
The progress bar has also been changed to have a subtle gradient between two shades of blue.

![ ](https://github.com/Jepl4r/hiby-mods/blob/unified_theme/themes/Unified%20Theme/Screenshots/Playing%20Page.png)

## Files page

Modified the SD-Card and OTG icons to have a modern look. (The USB icon that appears when connecting the device to a PC has also been changed)

![ ](https://github.com/Jepl4r/hiby-mods/blob/unified_theme/themes/Unified%20Theme/Screenshots/Files%20Page.png)

## TopBar Menu

Changed the brightness slider to be orange by adding an image and editing the respective file `hiby_pull_down_menu.view`
The Wi-Fi icon has also been changed to match the one from iOS. (The change is system wide)

![ ](https://github.com/Jepl4r/hiby-mods/blob/unified_theme/themes/Unified%20Theme/Screenshots/Pull%20Down%20Page.png)

## Boot Logo

Modified the logo image inside the `etc` and `boot_animation` folders to be orange/yellow.
(**Only** for the **Dark** theme)

If the images inisde the `etc` folder are not changed, the logo will appear white and then orange/yellow.

This is because the device uses the logo image inside the `etc` folder at boot and only after 1-3 seconds the one from the `boot_animation` folder is used.

![ ](https://github.com/Jepl4r/hiby-mods/blob/unified_theme/themes/Unified%20Theme/Screenshots/Logo.png)

# Difference between the two theme variants

The **Normal** theme is for use with the **hiby_player** binary that **doesn't** have the sorting patches.

The Sorted theme is for use with the **hiby_player** binary that **does** have the sorting patches.

The main difference is the scrollbar with letters that appears on the right side.

For the Normal variant the scrollbar starts with `A` and finishes with `#`

For the Sorted variant the scrollbar starts with `#` and finishes with `Z`

![ ](https://github.com/Jepl4r/hiby-mods/blob/unified_theme/themes/Unified%20Theme/Screenshots/Albums%20Page.png)
