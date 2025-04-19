## Icons

We'll need a set of icons to indicate weather, humidity, air quality, and other information. I've converted these icons from Google Fonts into BMP files, which we can use with the tinybmp crate.

We'll store the image bytes in a static array, allowing us to iterate through them and find the icon that matches the current weather.  To achieve this, I added a new function in `build.rs` that loads icon files from the specified directory and generates a static array in `icon.rs`. This way, we don't need to manually update the array each time a new icon is added.

## Modify build.rs file
In build.rs, we'll add a function that scans the `src/icons/` directory. You can grab the icons from this repo: https://github.com/ImplFerris/esp32-epaper-weather/tree/main/src/icons and place them into your own `src/icons` folder.

```rust
fn generate_icon_array() {
    let dest_path = Path::new("src/icons.rs");
    let icon_dir = Path::new("src/icons/");

    let mut generated_code = String::new();

    generated_code.push_str("pub static ICONS: &[(&str, &[u8])] = &[\n");

    if let Ok(entries) = fs::read_dir(icon_dir) {
        for entry in entries.flatten() {
            let path = entry.path();
            if let Some(file_name) = path.file_name().and_then(|f| f.to_str()) {
                generated_code.push_str(&format!(
                    "    (\"{}\", include_bytes!(\"{}/{}\")),\n",
                    file_name, "icons", file_name
                ));
            }
        }
    }

    generated_code.push_str("];\n");

    fs::write(dest_path, generated_code).expect("Failed to write icons.rs");

    println!("cargo:rerun-if-changed={}", icon_dir.to_str().unwrap());
}
```

Update the main function in build.rs to call our new function
```rust
fn main() {
    generate_icon_array();

    linker_be_nice();
    println!("cargo:rustc-link-arg=-Tdefmt.x");
    // make sure linkall.x is the last linker script (otherwise might cause problems with flip-link)
    println!("cargo:rustc-link-arg=-Tlinkall.x");
}
```

## icons.rs - Don't modify this manually!
If everything went smoothly, you should see the icons.rs file in the src folder with the content below. If the file wasn't created, try triggering the build script manually by running `cargo build`

```rust
pub static ICONS: &[(&str, &[u8])] = &[
    ("partly_cloudy_day.bmp", include_bytes!("icons/partly_cloudy_day.bmp")),
    ("nights_stay.bmp", include_bytes!("icons/nights_stay.bmp")),
    ("humidity_percentage.bmp", include_bytes!("icons/humidity_percentage.bmp")),
    ("cyclone.bmp", include_bytes!("icons/cyclone.bmp")),
    ("foggy.bmp", include_bytes!("icons/foggy.bmp")),
    ("snowing_heavy.bmp", include_bytes!("icons/snowing_heavy.bmp")),
    ("sunny.bmp", include_bytes!("icons/sunny.bmp")),
    ("cloud.bmp", include_bytes!("icons/cloud.bmp")),
    ("partly_cloudy_night.bmp", include_bytes!("icons/partly_cloudy_night.bmp")),
    ("rainy.bmp", include_bytes!("icons/rainy.bmp")),
    ("heat.bmp", include_bytes!("icons/heat.bmp")),
    ("clear_day.bmp", include_bytes!("icons/clear_day.bmp")),
    ("weather_mix.bmp", include_bytes!("icons/weather_mix.bmp")),
    ("weather_hail.bmp", include_bytes!("icons/weather_hail.bmp")),
    ("rainy_snow.bmp", include_bytes!("icons/rainy_snow.bmp")),
    ("mist.bmp", include_bytes!("icons/mist.bmp")),
    ("snowing.bmp", include_bytes!("icons/snowing.bmp")),
    ("rainy_heavy.bmp", include_bytes!("icons/rainy_heavy.bmp")),
    ("flood.bmp", include_bytes!("icons/flood.bmp")),
    ("sunny_snowing.bmp", include_bytes!("icons/sunny_snowing.bmp")),
    ("air.bmp", include_bytes!("icons/air.bmp")),
    ("storm.bmp", include_bytes!("icons/storm.bmp")),
    ("weather_snowy.bmp", include_bytes!("icons/weather_snowy.bmp")),
    ("thunderstorm.bmp", include_bytes!("icons/thunderstorm.bmp")),
    ("thermostat.bmp", include_bytes!("icons/thermostat.bmp")),
    ("rainy_light.bmp", include_bytes!("icons/rainy_light.bmp")),
];
```
