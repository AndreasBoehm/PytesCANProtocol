# Pytes CAN Protocol

A "reverse engineering" of the Pytes CAN protocol when used with Victron/Venus OS. Inspired by https://github.com/dfch/BydCanProtocol

The following information has been discovered between a Pytes V5α and a Victron Venus OS v3.34. 

* CAN protocol is set to 500kBit/s (no FD).
* All information has been dumped on a Raspberry 4 with Venus OS v3.34 (as written above) and a waveshare RS485 CAN HAT with `cansniffer -c -t 0 can0`.
* To test if the protocol works as expected you can use `cansend` on the Raspberry to send can messages to Venus OS. You need to send all messages, and do that in a loop, otherwise Venus OS does not show any information about the battery after some seconds.

## Pytes Communication / Identifiers
| Id  | Hex | Ascii | Description |
| --- | ----------------------- | -------- | --- |
| 351 | 38 02 E8 03 E8 03 C7 01 | 8....... | **Dvcc**: CVL, CCL, DCL, DVL |
|     |                         |          | [00:01] "38 02" (V/10, DeciVolt) 56.8V CVL |
|     |                         |          | [02:03] "E8 03" (A/10, DeciAmp) 100.0A CCL |
|     |                         |          | [04:05] "E8 03" (A/10, DeciAmp) 100.0A DCL |
|     |                         |          | [06:07] "C7 01" (V/10, DeciVolt) 45.5V DVL |
| 355 | 33 00 64 00             | 3.d.     | **StateInfo**: State of Charge, State of Health |
|     |                         |          | [00:01] "33 00" (%) 51% SoC |
|     |                         |          | [02:03] "64 00" (%) 100% SoH |
| 356 | 8E 14 F9 FF B4 00       | ......   | **BatteryStats**: Voltage, Amps, Temperature |
|     |                         |          | [00:01] "BE 14" (mV, MilliVolt) 52.62V current voltage |
|     |                         |          | [02:03] "F9 FF" (A/10, DeciAmp, signed) -0.7A consumed Amps; "-" discharge / "+" charge |
|     |                         |          | [04:05] "B4 00" (°C/10, Deci, signed) 18.0°C battery temperature |
| 35A | 00 00 00 00 00 00 00 00 | ........ | **Alarms and Warnings**, bit field, '0' means OK |
|     | 1. .. .. .. .. .. .. .. | ........ | [00,0] "1" Low battery voltage: Alarm |
|     | 4. .. .. .. .. .. .. .. | ........ | [00,0] "4" High temperature: Alarm |
|     | .4 .. .. .. .. .. .. .. | ........ | [00,1] "4" High battery voltage : Alarm |
|     | .. 1. .. .. .. .. .. .. | ........ | [01,0] "1" Low charge temperature: Alarm |
|     | .. 4. .. .. .. .. .. .. | ........ | [01,0] "4" High discharge current: Alarm |
|     | .. .1 .. .. .. .. .. .. | ........ | [01,1] "1" Low temperature: Alarm |
|     | .. .4 .. .. .. .. .. .. | ........ | [01,1] "4" High charge temperature: Alarm |
|     | .. .. 4. .. .. .. .. .. | ........ | [02,0] "4" Internal failure: Alarm |
|     | .. .. .1 .. .. .. .. .. | ........ | [02,1] "1" High charge current: Alarm |
|     | .. .. .. .1 .. .. .. .. | ........ | [03,1] "01" Cell imbalance: Alarm |
|     | .. .. .. .. 1. .. .. .. | ........ | [04,0] "1" Low battery voltage: Warning |
|     | .. .. .. .. 4. .. .. .. | ........ | [04,0] "4" High temperature: Warning |
|     | .. .. .. .. .4 .. .. .. | ........ | [04,1] "4" High battery voltage : Warning |
|     | .. .. .. .. .. 1. .. .. | ........ | [05,0] "1" Low charge temperature: Warning |
|     | .. .. .. .. .. 4. .. .. | ........ | [05,0] "4" High discharge current: Warning |
|     | .. .. .. .. .. .1 .. .. | ........ | [05,1] "1" Low temperature: Warning |
|     | .. .. .. .. .. .4 .. .. | ........ | [05,1] "4" High charge temperature: Warning |
|     | .. .. .. .. .. .. 4. .. | ........ | [06,0] "4" Internal failure: Warning |
|     | .. .. .. .. .. .. .1 .. | ........ | [06,1] "1" High charge current: Warning |
|     | .. .. .. .. .. .. .. .1 | ........ | [07,1] "1" Cell imbalance: Warning |
| 35E | 50 59 54 45 53          | PYTES    | **ManufacturerInfo** |
|     |                         |          | [00:04] "50 59 54 45 53" (string) "PYTES" manufacturer identification
| 35F | 01 00 6E 01 32 00       | ..n.2.   | **BatteryInfo**: Firmware version, Ah available |
|     |                         |          | [00:01] "01 00" (v'01'.'00') **maybe** v01.00 hardware version |
|     |                         |          | [02:03] "6E 01" (v'6E'.'01') v110.01 firmware version |
|     |                         |          | [04:05] "32 00" (Ah) 50Ah capacity available |
| 360 | 00                      | ........ | *???* always null |
| 372 | 02 00 01 00 01 00 02 00 | ........ | **BankInfo** |
|     |                         |          | [00:01] "02 00" 2 batteries online |
|     |                         |          | [02:03] "01 00" 1 batteries block charge |
|     |                         |          | [04:05] "00 00" 1 batteries block discharge |
|     |                         |          | [06:07] "02 00" 2 batteries offline |
| 373 | D8 0C DA 0C 21 01 23 01 | ....#.#. | **CellInfo**: Cell Voltage and Temperature |
|     |                         |          | [00:01] "D8 0C" (mV) 3.288V Lowest Cell Voltage, see 374 |
|     |                         |          | [02:03] "DA 0C" (mV) 3.290V Highest Cell Voltage, see 375 |
|     |                         |          | [04:05] "21 01" (Kelvin) 289K, 16°C Minimum Cell Temperature, see 376 |
|     |                         |          | [06:07] "23 01" (Kelvin) 291K, 18°C Maximum Cell Temperature, see 377 |
| 374 | 30 38 30 30 00 00 00 00 | 0800.... | **Battery/Cell name** (string) "0800 "with "Lowest Cell Voltage", see 373 |
| 375 | 30 34 30 30 00 00 00 00 | 0400.... | **Battery/Cell name** (string) "0400" with "Highest Cell Voltage", see 373 |
| 376 | 30 32 30 30 00 00 00 00 | 0000.... | **Battery/Cell name** (string) "0200" with "Minimum Cell Temperature", see 373 |
| 377 | 30 33 30 30 00 00 00 00 | 0000.... | **Battery/Cell name** (string) "0300" with "Maximum Cell Temperature", see 373 |
| 378 | 40 08 00 00 2B 07 00 00 | @...2... | **History**: Charged / Discharged Energy |
|     |                         |          | [00:03] "40 08 00 00" (kWh/10, HectoWattHour) 211.2kWh Charged Energy |
|     |                         |          | [04:07] "2B 07 00 00" (kWh/10, HectoWattHour) 183.5kWh Discharged Energy |
| 379 | 64 00                   | d.       | **BatterySize**: Installed Ah |
|     |                         |          | [00:01] "64 00" (Ah) 100Ah |
| 380 | XX XX XX XX XX XX XX XX | redacted | **Serial number - part 1** (string) |
| 381 | XX XX XX XX XX XX XX XX | redacted | **Serial number - part 2** (string) |

 
### How it looks like in Venus OS

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/c56bf0a1-7ccf-40f4-95c8-f68b0b0bf4fe)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/9696afd2-70c3-4fda-bd3b-fa6a4c7729f2)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/c2f9ffeb-64d4-4fd5-aef6-772cb88352a4)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/19f90cb4-7bab-4eff-af18-c545bc6fae49)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/2838e9da-771d-49ac-8c82-bce841b40821)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/22e2e31e-83d9-4480-aae5-5cd0cdc17085)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/02e2e040-a7a8-4c99-b8ad-1f65d4425f46)





