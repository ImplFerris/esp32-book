## Supported Characters

When referring to the HD44780 datasheet, you'll find two character set tables corresponding to two different ROM versions(A00 and A02). To determine which ROM your display uses, try unique characters from both tables. The one that displays correctly indicates the ROM version. Once identified, you only need to refer to the relevant table. 

In my case, the LCD module I'm using is based on ROM version A00. I'll present the A00 table and explain how to interpret it, though the interpretation logic is the same for both versions.

<img style="display: block; margin: auto;" alt="lcd1602" src="./images/lcd1602-characters-set.png"/>

It's an 8-bit character, where the upper 4 bits come first, followed by the lower 4 bits, to form the complete character byte. In the reference table, the upper 4 bits correspond to the columns, while the lower 4 bits correspond to the rows.

For example, to get the binary representation of the character "#," the upper 4 bits are 0010, and the lower 4 bits are 0011. Combining them gives the full binary value 00100011. In Rust, you can represent this value either in binary (0b00100011) or as a hexadecimal (0x23).
 
### hd44780-driver crate

In the `hd44780-driver` crate we are using, we can write characters directly as a single byte or a sequence of bytes.

#### Write single byte
```rust
lcd.write_byte(0x23, &mut timer).unwrap();
lcd.write_byte(0b00100011, &mut timer).unwrap();
```

#### Write multiple bytes
```rust
lcd.write_bytes(&[0x23, 0x24], &mut timer).unwrap();
```
