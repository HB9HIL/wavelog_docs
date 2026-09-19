# Logging

Clicking the QSO drop-down menu provides two options, **Live QSO** and **Post QSO**.

- Live QSO option has a real-time, per-second precision clock, and can also use information from the radio CAT interface to populate fields such as the frequency.
- Post QSO should be used to log contacts that don't need those fields mentioned above to be populated automatically.

![QSO Entry Area](https://github.com/user-attachments/assets/d8fbc10a-7dae-4238-9d65-6682bdbdc742)

After a callsign is entered, and the next field is selected, Wavelog will check qrz.com for details of the entered callsign and populate fields automatically with details if they are available.

## Previous contacts list

Below the map, the few previous contacts from your logbook are displayed. By default, last 5 contacts are displayed, but this number is configurable in the account settings.

## Entering the QSO data

<img src="https://github.com/wavelog/wavelog/assets/13950650/b74b1e7a-014a-4f53-abe2-35d33265e89d" alt="QSO data entry form">

### Quick QSY

!!! note "New in Wavelog 3.2.3"

With an active CAT connection (WebSocket or polling), you can change the frequency of your radio directly from the callsign field: type a number instead of a callsign and press `Tab`, `Space` or `Enter`.

The entry is interpreted in kHz and Wavelog understands it in two ways:

- **Offset entry** — if the number does not fall into an amateur band by itself, it is added to the integer-MHz part of the current frequency. On 7100 kHz, entering `134` QSYs to 7134 kHz, `34.5` to 7034.5 kHz. On 14230 kHz, entering `155` QSYs to 14155 kHz. This only works if the resulting frequency stays within the current band.
- **Full-frequency entry** — if the number itself is a frequency within an amateur band, the radio QSYs there, even across bands. On 7135 kHz, entering `14200` switches to 14.200 MHz.

Both dot and comma are accepted as decimal separator (`34.5` and `34,5` are equivalent).

After the QSY, the callsign field is cleared and the focus is returned to it. The frequency, band and mode displays follow the radio automatically, just like a VFO QSY. The current mode is kept; if no mode is set, LSB is used below 10 MHz and USB above.

The same feature is available in the [Contest Logging Engine](../contesting/logging.md#qso-logger).

### Date and time entry

While post-logging a QSO there is a shortcut for entering date and time: You can enter the date as "202109902" and it is automatically re-formatted to be "2021-09-02". Same goes for time. You can enter the time as "1413" and it is re-formatted to "14:13".

### RST - Readability Signal Tone

**RST (S)** is the code sent by you to your contact.
**RST (R)** is the code received by you from your contact.

For more information, please refer to the [R-S-T system](https://en.wikipedia.org/wiki/R-S-T_system) Wikipedia page.

### Satellite tab

The satellite tab enables satellite name and mode to be entered from drop down menus, this also populates the frequency, band and mode fields in the QSO tab.

### General tab

The general tab allows to add additional information like IOTA or SOTA reference or Sig and Sig Info for WWFF or POTA contacts.

### Edit a QSO

You'll want to edit a qso for a number of reasons like adding grid-square, or updating the QSLing information. This is possible by going to the "Logbook" section then clicking the icon on the right hand side of each QSO.

### Deleting a QSO

You cannot delete a QSO directly on the QSO page. However, you can delete a QSO by going to "Logbook" -> then selecting a QSO you want to edit, once in this section your displayed the option to "Delete QSO"
