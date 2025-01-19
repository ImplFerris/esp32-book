# RFID
In this section, we'll use the RFID Card Reader (RC522) module to read data from RFID tags and key fobs.

You've likely encountered these in everyday life; opening your apartment door with a smart key, accessing office doors with a tag, entering smart parking lots, or making payments with a contactless credit card. If you've used a hotel keycard or a toll pass in your car, you've already experienced RFID in action!

<img style="display: block; margin: auto;width:350px;" alt="MIFARE Memory layout" src="./images/rfid-card.tag.png"/>


RFID (Radio Frequency Identification) is a technology that uses radio waves to identify and track objects, animals. It wirelessly transmits the stored data from a tag (containing a chip and antenna) to a reader when in range.
 
## Categories By Range  
RFID systems can be categorized by their operating frequency. The three main types are:  

- **Low Frequency (LF)**: Operates at ~125 kHz with a short range (up to 10cm). It's slower and commonly used in access control and livestock tracking.  

- **High Frequency (HF)**: Operates at 13.56 MHz with a range of 10cm to 1m. It offers moderate speed and is widely used in access control systems, such as office spaces, apartments, hotel keycards, as well as in ticketing, payments, and data transfer.  RC522 module that we are going to use operates at 13.56MHz.

- **Ultra-High Frequency (UHF)**: Operates at 860–960 MHz with a range of up to 12m. It's faster and commonly used in retail inventory management, anti-counterfeiting, and logistics.

## Categories By Power source
RFID tags can either be active or passive, depending on how they are powered.

- **Active tags**: They have their own battery and can send signals on their own. These are typically used on large objects like rail cars, big reusable containers, and assets that need to be tracked over long distances.
- **Passive tags:**  Unlike active tags, passive tags don't have a battery. They rely on the electromagnetic fields emitted by the RFID reader to power up. Once energized, they transmit data using radio waves. These are the most common type of RFID tags and are likely the ones you've encountered in everyday life. If you guessed it correctly, yes the RC522 is the passive tags.

## Components:

RFID systems consist of an RFID Reader, technically referred to as the PCD (Proximity Coupling Device). In passive RFID tags, the reader powers the tag using an electromagnetic field. The tags themselves are called RFID Tags or, in technical terms, PICCs (Proximity Integrated Circuit Cards).  It is good to know its technical terms also, it will come in handy if you want to refer the datasheet and other documents.

Reader typically include memory components like FIFO buffers and EEPROM. They also incorporate cryptographic features to ensure secure communication with Tags, allowing only authenticated RFID readers to interact with them. For example, RFID readers from NXP Semiconductors use the Crypto-1 cipher for authentication.

Each RFID tag has a hardcoded UID (Unique Identifier), which can be 4, 7, or 10 bytes in size.
 

## References
- [What Is A RFID Antenna?](https://www.sannytelecom.com/what-is-a-rfid-antenna/)
- [Types of RFID Systems](https://www.impinj.com/products/technology/how-can-rfid-systems-be-categorized)

 


