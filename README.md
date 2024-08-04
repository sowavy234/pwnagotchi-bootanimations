# 🐾 Adding a Boot Animation to Pwnagotchi

Now you can add boot animations to your Pwnagotchi! I created this project while building a Cyberpunk-themed Pwnagotchi for myself. I will be releasing a complete set as soon as it's finished. This guide will walk you through the process step-by-step.

## Prerequisites

- 🐧 Pwnagotchi setup on Raspberry Pi
- 📟 A Supported Screen, list can be found [here](https://github.com/jayofelony/pwnagotchi/tree/master/pwnagotchi/ui/hw)
- 🔐 SSH access to your Pwnagotchi device

## Instructions

### 1. SSH into Your Pwnagotchi

Open a terminal and SSH into your Pwnagotchi device:

```bash
ssh pi@ip
```

### 2. Transfer the Boot Image

Transfer the boot image from your local machine to the Pwnagotchi device:

```bash
scp /path/to/your/frames/* pi@10.0.0.2:/home/pi/
```

### 3. Create the Directory and Move the Images

Create a directory for the boot animation images and move the images there:

```bash
sudo mkdir -p /usr/local/bin/boot_animation_images
sudo mv /home/pi/*.png /usr/local/bin/boot_animation_images/
```

### 4. Update the `fonts.py` File

Update the `fonts.py` file to ensure proper font settings, Directly copy-paste in terminal:

```bash
sudo tee /usr/local/lib/python3.11/dist-packages/pwnagotchi/ui/fonts.py > /dev/null << EOF
from PIL import ImageFont

# should not be changed
FONT_NAME = 'DejaVuSansMono'

# can be changed
STATUS_FONT_NAME = None
SIZE_OFFSET = 0

Bold = None
BoldSmall = None
BoldBig = None
Medium = None
Small = None
Huge = None

def init(config):
    global STATUS_FONT_NAME, SIZE_OFFSET
    STATUS_FONT_NAME = config.get('ui', {}).get('font', {}).get('name', FONT_NAME)
    SIZE_OFFSET = config.get('ui', {}).get('font', {}).get('size_offset', SIZE_OFFSET)
    setup(10, 8, 10, 25, 25, 9)

def status_font(old_font):
    global STATUS_FONT_NAME, SIZE_OFFSET
    if STATUS_FONT_NAME is None:
        STATUS_FONT_NAME = FONT_NAME  # Fallback to default font if not set
    return ImageFont.truetype(STATUS_FONT_NAME, size=old_font.size + SIZE_OFFSET)

def setup(bold, bold_small, medium, huge, bold_big, small):
    global Bold, BoldSmall, Medium, Huge, BoldBig, Small, FONT_NAME
    Small = ImageFont.truetype(FONT_NAME, small)
    Medium = ImageFont.truetype(FONT_NAME, medium)
    BoldSmall = ImageFont.truetype(f"{FONT_NAME}-Bold", bold_small)
    Bold = ImageFont.truetype(f"{FONT_NAME}-Bold", bold)
    BoldBig = ImageFont.truetype(f"{FONT_NAME}-Bold", bold_big)
    Huge = ImageFont.truetype(f"{FONT_NAME}-Bold", huge)
EOF
```

Sure, here's a more detailed explanation of what needs to be changed in the script, with specific instructions:

### 5. Create the Boot Animation Script

Create a script to display the boot animation. **Make sure to replace the import and the `initialize` function with the correct screen type you are using**:

The current code below is for the Waveshare 2.13 V4 screen, but you can change the screen type by following the guide below:

1. **Import the correct screen type**:
   - The line `from pwnagotchi.ui.hw import waveshare2in13_V4` should be replaced with the import corresponding to your specific screen type.

2. **Update the class definition**:
   - The class name `CustomWaveshareV4` and its parent class `waveshare2in13_V4.WaveshareV4` should be updated to match the correct screen type.

3. **Modify the `initialize` function**:
   - The line `from pwnagotchi.ui.hw.libs.waveshare.epaper.v2in13_V4.epd2in13_V4 import EPD` should be updated based on your screen type.

4. **Update the display type in the `config` dictionary**:
   - The line `'type': 'ws4'` should be updated to match your screen type.

Directly copy paste in terminal:

```bash
sudo tee /usr/local/bin/boot_animation.py > /dev/null << EOF
import time
from PIL import Image
from pwnagotchi.ui.hw import waveshare2in13_V4  # Change this import based on your screen type
import os

class CustomWaveshareV4(waveshare2in13_V4.WaveshareV4):  # Change this based on your screen type
    def __init__(self, config):
        super().__init__(config)

    def layout(self):
        return None

    def initialize(self):
        from pwnagotchi.ui.hw.libs.waveshare.epaper.v2in13_V4.epd2in13_V4 import EPD  # Change this based on your screen type
        self._display = EPD()
        self._display.init()
        self._display.Clear(0xFF)  # Clear to white

    def render(self, canvas):
        buf = self._display.getbuffer(canvas)
        self._display.displayPartial(buf)

    def clear(self):
        self._display.Clear(0xFF)  # Clear to white

def show_boot_animation():
    config = {
        'ui': {
            'display': {
                'type': 'ws4',  # Change this based on your screen type
                'color': 'black'
            }
        }
    }
    display = CustomWaveshareV4(config)
    display.initialize()
    width, height = 250, 122  # Dimensions of the 2.13 inch display

    # Path to the boot animation frames
    frames_path = '/usr/local/bin/boot_animation_images/'

    # Ensure at least 3 loops within the 5-second duration
    min_loops = 3
    total_duration = 5
    start_time = time.time()
    loop_count = 0
    
    try:
        while (time.time() - start_time < total_duration) or (loop_count < min_loops):
            # Get all PNG files in the directory
            frames = sorted([f for f in os.listdir(frames_path) if f.endswith('.png')])
            
            for frame in frames:
                if (time.time() - start_time >= total_duration) and (loop_count >= min_loops):
                    break
                with Image.open(os.path.join(frames_path, frame)) as img:
                    img = img.resize((width, height)).transpose(Image.FLIP_TOP_BOTTOM).transpose(Image.FLIP_LEFT_RIGHT).convert('1')  # Resize, flip vertically and horizontally, and convert to 1-bit color
                    display.render(img)
                    time.sleep(0.167)  # Adjust this value to control animation speed
            loop_count += 1
    except Exception as e:
        print(f"An error occurred: {str(e)}")
        display.clear()  # Only clear if an exception occurs

if __name__ == "__main__":
    show_boot_animation()
EOF
```

### 6. Make the Script Executable

Make the boot animation script executable:

```bash
sudo chmod +x /usr/local/bin/boot_animation.py
```

### 7. Test the Script

Run the boot animation script to test it:

```bash
sudo python3 /usr/local/bin/boot_animation.py
```

### 8. Edit the Pwnagotchi Launcher File

Edit the Pwnagotchi launcher file to include the boot animation script:

```bash
sudo tee /usr/bin/pwnagotchi-launcher > /dev/null << EOF
#!/usr/bin/env bash
source /usr/bin/pwnlib

# Run boot animation
/usr/bin/python3 /usr/local/bin/boot_animation.py

# we need to decrypt something
if is_crypted_mode; then
  while ! is_decrypted; do
    echo "Waiting for decryption..."
    sleep 1
  done
fi

if is_auto_mode; then
  /usr/local/bin/pwnagotchi
  systemctl restart bettercap
else
  /usr/local/bin/pwnagotchi --manual
fi
EOF
```

Make the launcher script executable:

```bash
sudo chmod +x /usr/bin/pwnagotchi-launcher
```

### 9. Reload the Systemd Manager Configuration

Reload the systemd manager configuration:

```bash
sudo systemctl daemon-reload
```

### 10. Restart Pwnagotchi

Restart Pwnagotchi to apply the changes:

```bash
sudo systemctl restart pwnagotchi
```

## 🎥 Boot Animation Showcase

https://www.reddit.com/r/pwnagotchi/comments/1ej5fsl/cyberpunk_rebecca_x_pwnagotchi_update_1/

Feel free to replace these placeholders with actual frames from your boot animation to give a visual preview.

Enjoy your new boot animation! 🎉 Your Pwnagotchi is now even more personalized and stylish.