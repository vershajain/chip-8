# chip-8
import random
import time

class CHIP8:
    def __init__(self, debug=False):
        self.debug = debug 
        self.memory = [0] * 4096 
        self.V = [0] * 16  
        self.I = 0  
        self.PC = 0x200  
        self.SP = 0  
        self.stack = [0] * 16  
        self.delay_timer = 0  
        self.sound_timer = 0 
        self.keys = [0] * 16  
        self.screen = [0] * (64 * 32)  
        self.previous_screen = self.screen[:]  

    def load_program(self, program):
        """Load a program into memory at 0x200"""
        for i in range(len(program)):
            self.memory[0x200 + i] = program[i]
        print(" Program loaded successfully!")

    def cycle(self):
        """Fetch-decode-execute cycle"""
        opcode = self.memory[self.PC] << 8 | self.memory[self.PC + 1]
        if self.debug:
            print(f"🔹 Executing opcode: {hex(opcode)} at PC: {hex(self.PC)}")

        self.execute_opcode(opcode)

        if opcode & 0xF000 != 0x1000:  # If not a jump instruction
            self.PC += 2

        if self.delay_timer > 0:
            self.delay_timer -= 1
        if self.sound_timer > 0:
            self.sound_timer -= 1

    def execute_opcode(self, opcode):
        """Decode and execute the opcode"""
        if opcode == 0x00E0:  # Clear the screen
            self.screen = [0] * (64 * 32)
        elif opcode & 0xF000 == 0x1000:  # Jump to address NNN
            self.PC = opcode & 0x0FFF
        elif opcode & 0xF000 == 0x6000:  # Set Vx to NN
            x = (opcode & 0x0F00) >> 8
            nn = opcode & 0x00FF
            self.V[x] = nn
        elif opcode & 0xF000 == 0x7000:  # Add NN to Vx
            x = (opcode & 0x0F00) >> 8
            nn = opcode & 0x00FF
            self.V[x] = (self.V[x] + nn) & 0xFF
        elif opcode & 0xF000 == 0xD000:  # Draw sprite at (Vx, Vy)
            x = self.V[(opcode & 0x0F00) >> 8] % 64
            y = self.V[(opcode & 0x00F0) >> 4] % 32
            height = opcode & 0x000F
            self.V[0xF] = 0  # Reset collision flag

            for row in range(height):
                sprite = self.memory[self.I + row]
                for col in range(8):
                    if (sprite & (0x80 >> col)) != 0:
                        index = (x + col + ((y + row) * 64)) % (64 * 32)
                        if self.screen[index] == 1:
                            self.V[0xF] = 1  # Collision detected
                        self.screen[index] ^= 1  # XOR drawing
        elif opcode & 0xF0FF == 0xF00A:  # FX0A: Wait for key press
            x = (opcode & 0x0F00) >> 8
            print(" Waiting for key press...")
            key_pressed = self.wait_for_keypress()
            self.V[x] = key_pressed
            print(f" Key pressed: {hex(key_pressed)}, Stored in V{x}: {hex(self.V[x])}")
            self.PC += 2  # Move forward

    def draw(self):
        """Render the screen only if it has changed"""
        if self.screen == self.previous_screen:
            return  # No change, so don't print

        self.previous_screen = self.screen[:]
        print(f"Current Value of V0: {hex(self.V[0])}")

        for row in range(32):
            print("".join("█" if self.screen[row * 64 + col] else "." for col in range(64)))
        print("\n" + "=" * 64)

    def wait_for_keypress(self):
        """Wait for a valid key press"""
        while True:
            key = self.handle_input()
            if key != -1:
                return key

    def handle_input(self):
        """Simulate keyboard input"""
        key = input("🔹 Press a key (1-9, A-F): ").lower()
        key_map = {
            '1': 0x1, '2': 0x2, '3': 0x3, '4': 0xC, 
            'q': 0x4, 'w': 0x5, 'e': 0x6, 'r': 0xD,
            'a': 0x7, 's': 0x8, 'd': 0x9, 'f': 0xE,
            'z': 0xA, 'x': 0x0, 'c': 0xB, 'v': 0xF
        }
        return key_map.get(key, -1)

def create_input_test_program():
   
    program = [
        0xF0, 0x0A,    # FX0A: Wait for key press, store in V0
        0x60, 0x01,  # 6001: Set V0 = 1 (confirm keypress works)
        0x00, 0xEE   # 00EE: Return (stop execution)
    ]
    with open('input_test.ch8', 'wb') as f:
        f.write(bytes(program))
def run_chip8():
    """Runs the CHIP-8 emulator"""
    create_input_test_program()
    chip8 = CHIP8(debug=False)

    program = open("input_test.ch8", "rb").read()
    chip8.load_program(list(program))

    while True:
        chip8.cycle()
        chip8.draw()
        time.sleep(1 / 60)

run_chip8()
