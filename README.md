import machine
from machine import Pin, SoftI2C, ADC, PWM
from i2c_lcd import I2cLcd  
import time
import dht
import network
import urequests
import ujson
import os
import socket
import _thread
from time import ticks_ms, ticks_diff


class IoTSystem:
    def __init__(self):
        # I2C LCD Configuration
        self.I2C_ADDR = 0x27
        self.I2C_NUM_ROWS = 4
        self.I2C_NUM_COLS = 20
        self.i2c = SoftI2C(scl=Pin(17), sda=Pin(16), freq=400000) 
        try:
            self.lcd = I2cLcd(self.i2c, self.I2C_ADDR, self.I2C_NUM_ROWS, self.I2C_NUM_COLS)
            print("LCD initialized successfully!")
        except Exception as e:
            print("Error initializing LCD:", e)
            self.lcd = None

    def main_loop(self):
        while True:
            self.check_button()
            current_time = ticks_ms()
            if self.system_active and self.wlan and self.wlan.isconnected():
                if ticks_diff(current_time, self.last_sensor_read) >= self.sensor_interval:
                    self.read_all_sensors()
                    self.check_limits_and_alert()
                    self.last_sensor_read = current_time

                if ticks_diff(current_time, self.last_data_send) >= self.data_interval:
                    if self.temp is not None and self.hum is not None:
                        self.send_to_thingspeak()
                        print("Data sent to ThingSpeak at", ticks_ms())
                    self.last_data_send = current_time

                if not self.menu_displayed:
                    self.lcd.clear()
                    self.lcd.move_to(0, 0)
                    self.lcd.putstr("1: SET LIMIT")
                    self.lcd.move_to(0, 1)
                    self.lcd.putstr("2: RECEIVE DATA")
                    self.lcd.move_to(0, 2)
                    self.lcd.putstr("3: SENSORS OPTIONS")
                    self.lcd.move_to(0, 3)
                    self.lcd.putstr("4: SEND FILE")
                    self.menu_displayed = True

                key = self.read_keypad()
                if key:
                    self.menu_displayed = False
                    if key == '1':
                        self.set_limits()
                    elif key == '2':
                        start_date = self.read_date("START DATE")
                        if start_date and self.system_active:
                            end_date = self.read_date("END DATE")
                            if end_date and self.system_active:
                                self.receive_from_thingspeak(start_date, end_date)
                    elif key == '3':
                        self.lcd.clear()
                        self.lcd.move_to(0, 0)
                        self.lcd.putstr("1: SENSOR DATA")
                        self.lcd.move_to(0, 1)
                        self.lcd.putstr("2: SENSOR LIMITS")
                        sub_key = None
                        while self.system_active and sub_key not in ['1', '2']:
                            sub_key = self.read_keypad()
                            time.sleep(0.01)
                            self.check_button()
                        if sub_key == '1' and self.system_active:
                            if self.temp is not None and self.hum is not None:
                                self.display_sensor_data()
                            else:
                                self.lcd.clear()
                                self.lcd.move_to(0, 1)
                                self.lcd.putstr("ERROR...")
                                time.sleep(1)
                        elif sub_key == '2' and self.system_active:
                            self.display_sensor_limits()
                    elif key == '4':
                        files = self.list_last_files()
                        if not files:
                            self.lcd.clear()
                            self.lcd.move_to(0, 1)
                            self.lcd.putstr("No files found")
                            time.sleep(1)
                        else:
                            self.lcd.clear()
                            for i, file in enumerate(files):
                                self.lcd.move_to(0, i)
                                self.lcd.putstr(f"{i+1}:{file[:17]}")
                            selected = None
                            valid_options = ['1', '2', '3', '4'][:len(files)]
                            while self.system_active and selected not in valid_options:
                                selected = self.read_keypad()
                                time.sleep(0.01)
                                self.check_button()
                            if selected and self.system_active:
                                file_to_send = files[int(selected) - 1]
                                self.lcd.clear()
                                self.lcd.move_to(0, 1)
                                self.lcd.putstr("File selected:")
                                self.lcd.move_to(0, 2)
                                self.lcd.putstr(file_to_send[:20])
                                time.sleep(2)
                                self.send_file_to_pc(self.wlan.ifconfig()[0], file_to_send)
                                self.menu_displayed = False
            time.sleep(0.01)

if __name__ == "__main__":
    iot_system = IoTSystem()
    iot_system.main_loop()
