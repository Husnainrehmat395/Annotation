# Annotation
import tkinter as tk
from tkinter import filedialog, messagebox, Frame, Label
from PIL import Image, ImageTk, ImageEnhance
import cv2
import os
import numpy as np

# Global variables
drawing = False
prev_point = None
thickness = 1  # starting pen width thickness
img_list = []
current_img_index = 0
img = None
output_folder = ""
img_display = None  # Keep a global reference to avoid garbage collection
history_stack = []  # Stack to keep track of image history for undo
zoom_scale = 1.0  # Scale for zooming
pan_start = None  # Starting point for panning
canvas_offset_x = 0  # Offset for panning
canvas_offset_y = 0  # Offset for panning
line_only_img = None  # Image to store the lines drawn without the background
image_saved = False  # Track whether the image has been saved

# Callback function for drawing on the canvas
def draw_line(event):
    global drawing, prev_point, img, img_display, thickness, history_stack, line_only_img, image_saved

    if drawing and prev_point is not None:
        # Save the current state before drawing the new line
        history_stack.append(img.copy())

        # Define line color
        color = (255, 0, 0)  # Red color

        # Adjust the coordinates for zoom and pan, to map the canvas coordinates back to the original image
        scaled_prev_point = (
            int((prev_point[0] - canvas_offset_x) / zoom_scale),
            int((prev_point[1] - canvas_offset_y) / zoom_scale)
        )
        scaled_current_point = (
            int((event.x - canvas_offset_x) / zoom_scale),
            int((event.y - canvas_offset_y) / zoom_scale)
        )

        # Ensure coordinates are within image bounds to avoid drawing outside
        img_height, img_width, _ = img.shape
        scaled_prev_point = (
            min(max(scaled_prev_point[0], 0), img_width - 1),
            min(max(scaled_prev_point[1], 0), img_height - 1)
        )
        scaled_current_point = (
            min(max(scaled_current_point[0], 0), img_width - 1),
            min(max(scaled_current_point[1], 0), img_height - 1)
        )

        # Draw the line on the original image using OpenCV
        cv2.line(img, scaled_prev_point, scaled_current_point, color, thickness, lineType=cv2.LINE_AA)

        # Draw the line on the "line-only" grayscale image, setting the pixel values to 255 (white)
        cv2.line(line_only_img, scaled_prev_point, scaled_current_point, 255, thickness, lineType=cv2.LINE_AA)

        # Update the display
        update_image()

        # Update the previous point
        prev_point = (event.x, event.y)

        # Mark image as unsaved
        image_saved = False

def start_drawing(event):
    global drawing, prev_point
    drawing = True
    prev_point = (event.x, event.y)

def stop_drawing(event):
    global drawing
    drawing = False

def update_thickness(val):
    global thickness
    thickness = int(val)  # Make sure the thickness is always an integer
    thickness_entry.delete(0, tk.END)
    thickness_entry.insert(0, str(thickness))

def set_thickness_from_entry():
    global thickness
    try:
        value = int(thickness_entry.get())  # Ensure the value is an integer
        if 1 <= value <= 50:  # Only allow values between 1 and 50
            thickness = value
            thickness_slider.set(thickness)
        else:
            messagebox.showerror("Error", "Please enter a value between 1 and 50.")
    except ValueError:
        messagebox.showerror("Error", "Invalid input. Please enter an integer value.")

def load_images():
    global img_list, current_img_index, output_folder, history_stack

    # Open file dialog to select a directory
    folder_path = filedialog.askdirectory()
    if folder_path:
        # Get all image files in the selected directory
        img_list = [os.path.join(folder_path, f) for f in os.listdir(folder_path) if f.lower().endswith(('.png', '.jpg', '.jpeg'))]
        if img_list:
            current_img_index = 0
            load_current_image()
        else:
            messagebox.showerror("Error", "No images found in the selected folder.")
            return

    # Prompt for output folder to save annotated images
    output_folder = filedialog.askdirectory(title="Select Output Folder to Save Annotated Images")
    if not output_folder:
        messagebox.showerror("Error", "No output folder selected.")
        return

def load_current_image():
    global img, img_display, canvas, img_list, current_img_index, history_stack, line_only_img, image_saved

    # Load the image using OpenCV
    img = cv2.imread(img_list[current_img_index])
    img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)  # Convert to RGB for displaying with Pillow

    # Resize if necessary to fit the window
    img = cv2.resize(img, (800, 600))

    # Clear history stack
    history_stack = []

    # Reset canvas offset
    global canvas_offset_x, canvas_offset_y
    canvas_offset_x = 0
    canvas_offset_y = 0

    # Initialize a blank grayscale image with zeros for storing the line (0 values mean no pixel)
    line_only_img = np.zeros((img.shape[0], img.shape[1]), dtype=np.uint8)

    # Update the display
    update_image()

    # Update file name label
    update_file_name_label()

    # Reset the saved state
    image_saved = True

def update_image():
    global img, img_display, canvas, zoom_scale, canvas_offset_x, canvas_offset_y

    # Convert the OpenCV image to ImageTk
    img_pil = Image.fromarray(img)
    
    # Enhance the brightness/intensity if needed (optional step)
    enhancer = ImageEnhance.Brightness(img_pil)
    img_pil = enhancer.enhance(1.2)  # Increase brightness by 20%

    # Apply zoom scaling
    width, height = img_pil.size
    img_pil = img_pil.resize((int(width * zoom_scale), int(height * zoom_scale)), Image.Resampling.LANCZOS)

    img_display = ImageTk.PhotoImage(img_pil)

    # Update the canvas
    canvas.delete("all")  # Clear previous image
    canvas.create_image(canvas_offset_x, canvas_offset_y, anchor=tk.NW, image=img_display)
    canvas.config(scrollregion=canvas.bbox(tk.ALL))  # Adjust scroll region based on new image size
    canvas.image = img_display  # Keep a reference to avoid garbage collection

def update_file_name_label():
    global img_list, current_img_index, file_name_label
    base_filename = os.path.basename(img_list[current_img_index])
    file_name_label.config(text=f"Annotating: {base_filename}")

def save_current_image():
    global img, img_list, current_img_index, output_folder, line_only_img, image_saved

    if img is not None and output_folder:
        # Save the annotated image
        img_bgr = cv2.cvtColor(img, cv2.COLOR_RGB2BGR)
        base_filename = os.path.basename(img_list[current_img_index])
        output_path = os.path.join(output_folder, base_filename)
        cv2.imwrite(output_path, img_bgr)
        messagebox.showinfo("Info", f"Image saved successfully: {output_path}")

        # Save the line-only image (grayscale)
        line_only_output_path = os.path.join(output_folder, f"line_only_{base_filename}")
        cv2.imwrite(line_only_output_path, line_only_img)
        messagebox.showinfo("Info", f"Line-only image saved successfully: {line_only_output_path}")

        # Mark image as saved
        image_saved = True

def next_image():
    global current_img_index, img_list, image_saved

    if img_list and current_img_index < len(img_list) - 1:
        if not image_saved:
            response = messagebox.askyesno("Unsaved Changes", "You have unsaved changes. Do you want to continue without saving?")
            if not response:
                return
        # Move to the next image
        current_img_index += 1
        load_current_image()
    elif current_img_index == len(img_list) - 1:
        messagebox.showinfo("Info", "All images have been annotated.")

def previous_image():
    global current_img_index, image_saved

    if img_list and current_img_index > 0:
        if not image_saved:
            response = messagebox.askyesno("Unsaved Changes", "You have unsaved changes. Do you want to continue without saving?")
            if not response:
                return
        # Move to the previous image
        current_img_index -= 1
        load_current_image()

def ctrl_z(event):
    global history_stack, img

    if history_stack:
        # Revert to the last saved state
        img = history_stack.pop()
        update_image()
    else:
        messagebox.showinfo("Info", "Nothing to undo.")

def zoom(event):
    global zoom_scale
    # Zoom in or out
    if event.delta > 0:
        zoom_scale *= 1.1  # Zoom in
    elif event.delta < 0:
        zoom_scale *= 0.9  # Zoom out

    update_image()  # Update the image display with the new zoom level

def start_pan(event):
    global pan_start
    pan_start = (event.x, event.y)  # Store the starting position when the middle mouse button is pressed

def pan_image(event):
    global pan_start, canvas_offset_x, canvas_offset_y

    if pan_start is not None:
        # Calculate the movement delta
        dx = event.x - pan_start[0]
        dy = event.y - pan_start[1]

        # Update the canvas offset based on the movement
        canvas_offset_x += dx
        canvas_offset_y += dy

        # Update the display
        update_image()

        # Update the pan starting point
        pan_start = (event.x, event.y)

def stop_pan(event):
    global pan_start
    pan_start = None  # Reset the starting point when the middle mouse button is released

# Create the main window
root = tk.Tk()
root.title("Image Annotator with Adjustable Line Thickness, Zoom, and Pan")

# Frame for holding the canvas and control panel
frame = Frame(root)
frame.pack(fill=tk.BOTH, expand=True)

# Create the Canvas for image display
canvas = tk.Canvas(frame, width=800, height=600, bg="white")
canvas.grid(row=0, column=0, padx=10, pady=10)

# Right panel for controls (similar to LabelImg)
control_panel = Frame(frame, width=200)
control_panel.grid(row=0, column=1, sticky=tk.N)

# Load images button
load_btn = tk.Button(control_panel, text="Load Images Folder", command=load_images, width=20)
load_btn.grid(row=0, column=0, pady=5)

# Save current image button
save_btn = tk.Button(control_panel, text="Save Current Image", command=save_current_image, width=20)
save_btn.grid(row=1, column=0, pady=5)

# Create a slider to adjust the thickness
thickness_slider = tk.Scale(control_panel, from_=1, to=50, resolution=1, orient=tk.HORIZONTAL, label="Line Thickness", command=update_thickness)
thickness_slider.grid(row=2, column=0, pady=5)

# Entry to manually input thickness
thickness_entry = tk.Entry(control_panel, width=10)
thickness_entry.grid(row=3, column=0, pady=5)
thickness_entry.insert(0, str(thickness))

# Button to update thickness from entry
set_thickness_btn = tk.Button(control_panel, text="Set Thickness", command=set_thickness_from_entry, width=20)
set_thickness_btn.grid(row=4, column=0, pady=5)

# Navigation buttons
prev_btn = tk.Button(control_panel, text="Previous Image", command=previous_image, width=20)
prev_btn.grid(row=5, column=0, pady=5)

next_btn = tk.Button(control_panel, text="Next Image", command=next_image, width=20)
next_btn.grid(row=6, column=0, pady=5)

# File name label to display the current image name
file_name_label = Label(frame, text="", font=("Helvetica", 14))
file_name_label.grid(row=1, column=0, padx=10, pady=5, sticky=tk.W)

# Bind mouse events to the canvas
canvas.bind("<ButtonPress-1>", start_drawing)
canvas.bind("<ButtonRelease-1>", stop_drawing)
canvas.bind("<B1-Motion>", draw_line)

# Bind Ctrl + Z for undo
root.bind_all("<Control-z>", ctrl_z)
root.bind_all("<Control-Z>", ctrl_z)

# Bind mouse wheel event for zooming
canvas.bind("<MouseWheel>", zoom)

# Bind middle mouse button events for panning
canvas.bind("<ButtonPress-2>", start_pan)  # Middle mouse button press to start panning
canvas.bind("<B2-Motion>", pan_image)      # Middle mouse movement for panning
canvas.bind("<ButtonRelease-2>", stop_pan) # Release middle mouse button to stop panning

# Run the main loop
root.mainloop()
