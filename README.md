# subghz
## Reverse Engineering the CREATE Ceiling Fan RF protocol with Flipper Zero

The CREATE Ceiling fan remote uses 433.92MHz AM to communicate with the fan. I couldn't find an FCC ID or any other meaningful ID stamped on the remote, so I used Flipper's Frequency Analyzer to monitor whilst pressing buttons on the remote.

In my current location, there are 4 fans and 4 remotes, and each remote only controls the fan it's paired with. There doesn't seem to be any interactive method to pair a remote with a fan and so each fan must be statically paired with the remote that it's bundled with.

Using the Sub-GHz RAW capture feature, I can capture a signal from the remote and replay it multiple times and so we can assume that there's no dynamic or rolling code generation taking place. We wouldn't expect there to be on a ceiling fan communication protocol.

After capturing the RAW signal for the light on/off button, I upload the file to https://lab.flipp.dev/pulse-plotter so that I can visualize it.

We see a series of 30 pulses for each button press, of two separate sizes. The short pulses represent 0 and the long pulses represent 1. 

The short pulses are on for roughly 380us and off for 1080us. The long pulses are reversed.

First, we see a short pulse (380us) followed by roughly 5130us of silence before the command is issued.

As there's only 30 pulses for a button press, we can write out the bitstream manually: `111000011011111001110101101010`

Now, I'll capture the button press for the other 10 buttons on the remote. The first 20 bits don't change between button presses, but the second 10 bits do:

```
11100001011000010110 0100101011 #timer
11100001011000010110 1000010111 #wave
11100001011000010110 1001010110 #switch
11100001011000010110 1100100011 #fan
11100001011000010110 1111010000 #1
11100001011000010110 1110010001 #2
11100001011000010110 1101010010 #3
11100001011000010110 1100010011 #4
11100001011000010110 1011010100 #5
11100001011000010110 1010010101 #6
11100001011000010110 0101101010 #light
```

As I discovered earlier, this remote only controls the one lamp, so it's safe to assume that the unit ID is contained somewhere within the first 20 bits. Let's grab the other 3 remotes and find out.

Here is the bitstream of the timer button on all 4 remotes side by side, starting with the one above:

```
11100001 011000010110 0100101011
11100001 101001010111 0100101011
11100001 100110000101 0100101011
11100001 101111100111 0100101011
```

I've now inserted a second gap in the bitstream that shows the part that's changing between different remotes.

We now know that the first 8 bits seems to be fixed and is a preamble, the next 12 bits represents the unit ID and the last 10 bits represents the command.

12 bits representing the unit ID give us a total of 2^12 = 4096 different unit IDs which is easily brute-forceable within a reasonable timeframe.

Let's write a python script to brute force the light button and produce 4096 separate bit streams for the light command, one for each unit ID. We'll batch them into 256 per file, and also create a playlist file of them all that can be used with the Sub-GHz Playlist app:

```
#!/usr/bin/python3

id_bits = 12
per_file = 256
playlist_name = "BtnLight"
output_dir = "out/"

cmd = "0101101010" #cmd light
#cmd = "0100101011" #cmd timer
#cmd = "1000010111" #cmd wave
#cmd = "1001010110" #cmd switch
#cmd = "1100100011" #cmd fan
#cmd = "1111010000" #cmd 1
#cmd = "1110010001" #cmd 2
#cmd = "1101010010" #cmd 3
#cmd = "1100010011" #cmd 4
#cmd = "1011010100" #cmd 5
#cmd = "1010010101" #cmd 6

preamble = "11100001"

#-----#

playlist_output = ""
playlist_entry = "sub: /ext/subghz/Ceiling_Fans/CREATE_Spain_LH/" + playlist_name + "/" + str(per_file) + "/"
playlist_file = "out/playlist.txt"
fh_playlist = open(playlist_file, "w")
header = """
Filetype: Flipper SubGhz RAW File
Version: 1
Frequency: 433920000
Preset: FuriHalSubGhzPresetOok270Async
Protocol: RAW
"""

str_output = ""

fh_subfile = None
for i in range(pow(2, id_bits)):
    if (i % per_file == 0):
        if fh_subfile is not None:
            fh_subfile.close()
        start = i
        end = i + per_file - 1
        subfile_name = str(start) + "_" + str(end) + ".sub"
        playlist_output = playlist_entry + subfile_name + "\n"
        fh_playlist.write(playlist_output)
        fh_subfile = open(output_dir + subfile_name, "w")
        fh_subfile.write(header)

    bits = bin(int(str(i), base=10)).split("b")[1]
    pad = "0" * (id_bits - len(bits))
    unit_id = pad + bits
    s = preamble #fixed
    s += unit_id #device 12-bit
    s += cmd

    str_output = "RAW_Data: 382 -5136 "
    l = len(s)
    for n in range(0, l):
        if s[n] == "0":
            str_output += "382 "
            if n != (l-1):
                str_output += "-1026 "
        if s[n] == "1":
            str_output += "1026 "
            if n != (l-1):
                str_output += "-382 "
    str_output += "-12000\n"
    fh_subfile.write(str_output)
fh_playlist.close()
```

We can now bruteforce the entire 4096 space by playing the full playlist within roughly 4 minutes.

You can also create two sets of data for each command. I created a set of 16 sub files each containing 256 commands as well as a set of 64 sub files each containing 32 commands. Once you've narrowed down the unit ID using the 16 sub files containing 256 commands each, you can then narrow it down further. It takes 15 seconds to play a sub file containing 256 commands, and it takes 2.4 seconds to play a sub file containing 32 commands.
