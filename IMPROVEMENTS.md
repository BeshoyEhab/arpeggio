# Arpeggio Piano - Enhancement Guide

This document outlines how to implement several key features to make the Arpeggio application more realistic and feature-rich.

## 1. Sustain Pedal Functionality

On a real piano, the sustain pedal allows notes to continue ringing out even after the key is released. We can simulate this using the spacebar.

### Implementation Steps

1.  **Track Sustained Notes:** We need a new list to keep track of notes that should be sustained.

    In `main.py`, add a new list and a boolean to track the pedal state:

    ```python
    # ... existing variables
    sustain_pedal_down = False
    sustained_notes = []
    # ...
    ```

2.  **Handle Pedal Press/Release:** In the main event loop, listen for `KEYDOWN` and `KEYUP` events for the spacebar.

    ```python
    # Inside the main while loop's event handling section
    if event.type == pygame.KEYDOWN:
        if event.key == pygame.K_SPACE:
            sustain_pedal_down = True

    if event.type == pygame.KEYUP:
        if event.key == pygame.K_SPACE:
            sustain_pedal_down = False
            # When the pedal is released, stop all sustained notes
            for channel in sustained_notes:
                channel.fadeout(FADEOUT_TIME)
            sustained_notes.clear()
    ```

3.  **Modify Note Release Logic:** Change the `KEYUP` event handler for notes. If the sustain pedal is down, instead of stopping the note immediately, add its channel to the `sustained_notes` list.

    ```python
    # In the KEYUP event handler
    if note_name and note_name in active_keyboard_notes:
        channel = active_keyboard_notes.pop(note_name)
        if sustain_pedal_down:
            sustained_notes.append(channel)
        else:
            channel.fadeout(FADEOUT_TIME)
        # ... rest of the keyup logic
    ```

## 2. More Expressive Velocity

A real piano plays louder or softer depending on how hard a key is pressed. Since a standard keyboard doesn't have pressure sensitivity, we can map the Y-position of the mouse click on a key to the velocity.

### Implementation Steps

1.  **Update Mouse Click Handling:** In the `MOUSEBUTTONDOWN` event handler, get the `event.pos` and calculate the velocity based on the click's Y-coordinate relative to the key's height.

    ```python
    # In the MOUSEBUTTONDOWN event handler
    if event.type == pygame.MOUSEBUTTONDOWN:
        black_key = False
        for i, key in enumerate(black_keys):
            if key.collidepoint(event.pos):
                # Y-position relative to the key
                relative_y = event.pos[1] - key.y
                # Normalize to a 0-1 range, then scale to 0-127
                velocity = int((relative_y / key.height) * 127)
                play_note_with_limiter(black_sounds[i], velocity)
                black_key = True
                active_blacks.append([i, 30])
        for i, key in enumerate(white_keys):
            if key.collidepoint(event.pos) and not black_key:
                relative_y = event.pos[1] - key.y
                velocity = int((relative_y / key.height) * 127)
                play_note_with_limiter(white_sounds[i], velocity)
                active_whites.append([i, 30])
    ```

## 3. GUI for MIDI File Selection

Currently, the MIDI file is hardcoded. A file dialog would allow users to select their own MIDI files to play.

### Implementation Steps

1.  **Import `tkinter`:** `tkinter` is part of Python's standard library and can be used to create a simple file dialog.

    ```python
    # At the top of main.py
    import tkinter as tk
    from tkinter import filedialog
    ```

2.  **Create a Root Window and Hide It:** We only need the dialog, not a full `tkinter` GUI.

    ```python
    # Near the top of main.py
    root = tk.Tk()
    root.withdraw() # Hides the small root window
    ```

3.  **Create a "Load MIDI" Button/Key:** Let's use a key press (e.g., `K_l`) to open the file dialog.

    ```python
    # In the KEYDOWN event handler
    if event.key == pygame.K_l:
        # Open file dialog
        midi_file_path = filedialog.askopenfilename(
            initialdir="assets/MIDI/",
            title="Select a MIDI File",
            filetypes=(("MIDI Files", "*.mid"), ("All files", "*.*"))
        )
        if midi_file_path: # If a file was selected
            if load_midi_file(midi_file_path):
                print("Starting playback...")
                playback_active = True
                playback_start_time = pygame.time.get_ticks()
                current_msg_index = 0
            else:
                print(f"Could not play {midi_file_path}")
    ```
    *Note: You can remove the old hardcoded `K_KP_0` logic.*

## 4. Visual MIDI Playback (Falling Notes)

This feature, inspired by apps like Synthesia, provides a visual guide for learning songs by showing notes as they are about to be played.

### Implementation Steps

1.  **Define Note Colors:** Choose colors for the falling notes.

    ```python
    # At the top of main.py
    WHITE_NOTE_COLOR = (0, 255, 255) # Cyan
    BLACK_NOTE_COLOR = (255, 0, 255) # Magenta
    ```

2.  **Create a `draw_falling_notes` Function:** This function will render the notes that are coming up in the MIDI sequence.

    ```python
    def draw_falling_notes(now_ms):
        # Look ahead in the playback messages (e.g., 4 seconds)
        look_ahead_ms = 4000
        visible_notes = []
        for i in range(current_msg_index, len(playback_messages)):
            start_ms, duration_ms, index, note_type, velocity = playback_messages[i]
            if start_ms > now_ms + look_ahead_ms:
                break # Stop if the note is too far in the future
            if start_ms >= now_ms:
                visible_notes.append(playback_messages[i])

        for start_ms, duration_ms, index, note_type, velocity in visible_notes:
            time_to_impact = start_ms - now_ms
            # Calculate Y position based on time to impact
            # The note should be at the top of the piano (y=100) when time_to_impact is look_ahead_ms
            # and at the bottom (y=HEIGHT-300) when time_to_impact is 0
            y_pos = 100 + ((look_ahead_ms - time_to_impact) / look_ahead_ms) * (HEIGHT - 400)

            # Calculate X position and width
            if note_type == 'white':
                x_pos = index * 35
                width = 35
                color = WHITE_NOTE_COLOR
            else: # black
                # This requires a more complex mapping from black note index to screen position
                # For now, a simplified calculation:
                skip_count = (index // 5) * 2 + ((index % 5) > 1) + ((index % 5) > 3)
                x_pos = 23 + (index * 35) + (skip_count * 35)
                width = 24
                color = BLACK_NOTE_COLOR

            # Height is proportional to duration
            height = (duration_ms / 1000) * 50 # 50 pixels per second of duration

            pygame.draw.rect(screen, color, [x_pos, y_pos, width, height])
    ```

3.  **Call the Function in the Main Loop:** Before drawing the piano, draw the falling notes.

    ```python
    # In the main while loop, before draw_piano()
    if playback_active:
        draw_falling_notes(now_ms)

    white_keys, black_keys, active_whites, active_blacks = draw_piano(
        active_whites, active_blacks
    )
    # ...
    ```
    *Note: The `x_pos` calculation for black notes in this example is simplified. A more accurate approach would be to pre-calculate the exact `x` coordinate for each black key and store it in a list or dictionary.*
