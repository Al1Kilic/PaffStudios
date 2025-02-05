# PaffStudios
Onedio Personality Test
# Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
# .\chatbot\Scripts\Activate


import locale
import os

# Set locale to a generic one supported by most systems
locale.setlocale(locale.LC_ALL, 'en_US.UTF-8')
os.environ['LC_ALL'] = 'en_US.UTF-8'

import tkinter as tk
from tkinter import messagebox, ttk
from ttkbootstrap import Style
import requests
import torch
from langchain_ollama import OllamaLLM
import re
from deep_translator import GoogleTranslator 
import io
from PIL import Image, ImageTk
import json
from collections import Counter
from diffusers import StableDiffusionXLImg2ImgPipeline
import nltk
nltk.download("stopwords")
from nltk.corpus import stopwords
from nltk.stem.porter import PorterStemmer
from diffusers import StableDiffusionImg2ImgPipeline
from deep_translator import GoogleTranslator

UNSPLASH_ACCESS_KEY = '1cljjfbYrm0J9H2fuP7SfMjJ2aA2mB8E5QYzux4f5SU'

PERSONALITY_MAPPING = {
    1: ["relaxed", "social", "active", "reflective"],
    2: ["adventurous", "creative", "relaxed", "energetic"],
    3: ["leader", "listener", "joker", "observer"],
    4: ["direct", "cautious", "guided", "avoidant"],
    5: ["intuitive", "rational", "collaborative", "deliberate"],
    6: ["adventurous", "compassionate", "ambitious", "stable"],
    7: ["proactive", "planner", "collaborative", "observer"],
    8: ["growth", "balanced", "defensive", "focused"],
    9: ["romantic", "trustworthy", "independent", "social"],
    10: ["passionate", "supportive", "adventurous", "stable"]
}

# AI Model Initialization
model = OllamaLLM(model = "llama3")

model_id = "stabilityai/stable-diffusion-xl-refiner-1.0"
img2img_pipe = StableDiffusionXLImg2ImgPipeline.from_pretrained(model_id, torch_dtype=torch.bfloat16, variant="fp16")
img2img_pipe = img2img_pipe.to("cpu")

def translate_to_turkish(data):
    translator = GoogleTranslator(source='en', target='tr')
    for item in data:
        item["question"] = translator.translate(item.get("question", ""))
        if "options" in item:
            item["options"] = [translator.translate(option) for option in item["options"]]
    return data

# Generate personality-test questions using AI
def generate_personality_test():
    raw_output = model.invoke(input="Create me a Buzzfeed-like personality test. Test should have 10 questions and 4 options each. Make the test in JSON format.")

    # Extract JSON block using regex
    try:
        json_match = re.search(r'\[\s*{.*?}\s*\]', raw_output, re.DOTALL)
        if json_match:
            json_str = json_match.group(0)  # Extract JSON block
            print("Extracted JSON:", json_str)

            # Parse JSON string
            personality_data = json.loads(json_str)
            if isinstance(personality_data, list):
                # Translate questions and options into Turkish
                return translate_to_turkish(personality_data)
            else:
                raise ValueError("Extracted JSON is not a list.")
    except (json.JSONDecodeError, ValueError) as e:
        print("Error parsing AI output:", e)

    # Return empty list if parsing fails
    return []

def rewrite_query_for_unsplash(question):
    """
    Rewrite the given question into a concise query suitable for Unsplash searches.
    """
    # Prompt AI to rewrite the question as a concise query
    prompt = f"""
    Rewrite the following question into a concise but descriptive photo search query.
    Ensure the result includes multiple related keywords that help find a relevant image.
    
    Examples:
    - "What is your ideal weekend activity?" → "weekend activities, hiking, friends outdoors"
    - "How do you spend a rainy day?" → "rainy day, reading book, cozy home"
    - "What is your favorite type of vacation?" → "beach vacation, tropical, sunset, travel"
    
    Now, process this question: '{question}'
    """
    rewritten_prompt = model.invoke(input=prompt).strip()
    
    # Ensure the rewritten query is concise (under 50 characters)
    if len(rewritten_prompt) > 50:
        rewritten_prompt = rewritten_prompt[:47] + "..."  # Truncate and add ellipsis

    return rewritten_prompt

def fetch_image(query):
    """
    Fetch an image from Unsplash based on the given query.
    """
    # Fallback image placeholder (in case Unsplash returns irrelevant results)
    fallback_image_path = "fallback_image.jpg"  # Ensure this file is in your working directory
    
    # Perform Unsplash API request
    try:
        url = f'https://api.unsplash.com/photos/random?query={query}&client_id={UNSPLASH_ACCESS_KEY}'
        response = requests.get(url)

        if response.status_code == 200:
            img_url = response.json().get('urls', {}).get('regular')
            if img_url:
                img_data = requests.get(img_url).content
                return Image.open(io.BytesIO(img_data))
        
        print("Unsplash returned no relevant image. Using fallback.")
        return Image.open(fallback_image_path)  # Return fallback image if query fails
    except Exception as e:
        print(f"Error fetching image: {e}. Using fallback image.")
        return Image.open(fallback_image_path)

def generate_ai_image(prompt, num_attempts=3):
    """
    Fetch a real image based on the prompt, then modify it using Stable Diffusion Img2Img.
    """
    for attempt in range(num_attempts):
        try:
            # Fetch a real image from Unsplash
            real_image = fetch_image(prompt)
            if not real_image:
                raise Exception("Failed to fetch image from Unsplash.")

            # Resize the image to match the expected input size
            real_image = real_image.resize((256, 256), Image.LANCZOS)

            # Use Stable Diffusion to modify the image
            result_image = img2img_pipe(
                prompt=f"An artistic representation of {prompt}",
                image=real_image,
                strength=0.5,
                num_inference_steps=50,
                guidance_scale=9.0
            ).images[0]

            return result_image
        except Exception as e:
            print(f"Attempt {attempt + 1} failed: {str(e)}")
            if attempt == num_attempts - 1:
                raise

personality_quiz_data = generate_personality_test()

# Fallback data if AI fails to generate questions
if not personality_quiz_data:
    personality_quiz_data = [
        {
            "question": "What is your ideal weekend activity?",
            "options": [
                "Stay at home and read",
                "Go to a big party",
                "Explore a new hiking trail",
                "Plan your next big project"
            ]
        },
        {
            "question": "Which type of movie do you enjoy the most?",
            "options": [
                "Comedy",
                "Drama",
                "Horror",
                "Sci-Fi"
            ]
        }
    ]
    personality_quiz_data = translate_to_turkish(personality_quiz_data)

# Functions for the quiz app
def show_question():
    """
    Displays the current question and choices, along with an appropriate image.
    """
    try:
        # Fetch the current question data
        question_data = personality_quiz_data[current_question]

        # Update question text
        qs_label.config(text=question_data.get("question", "No question provided"))

        # Get the options (formerly misread as choices)
        options = question_data.get("options", [])
        if not options:
            raise ValueError(f"No options found for question: {question_data}")

        # Update buttons with the options
        for i, choice_text in enumerate(options):
            choice_btns[i].config(text=choice_text, state="normal")

        # Rewrite query for Unsplash and fetch image
        rewritten_query = rewrite_query_for_unsplash(question_data["question"])        
        # Fetch and display image
        image = fetch_image(rewritten_query)
        if image:
            resized_image = ImageTk.PhotoImage(image.resize((500, 300), Image.LANCZOS))
            image_label.config(image=resized_image)
            image_label.image = resized_image
        else:
            image_label.config(image="", text="No image available")

        # Disable the Next button until an answer is selected
        next_btn.config(state="disabled")
    except Exception as e:
        print(f"Error displaying question: {e}")
        messagebox.showerror("Error", f"Failed to display question: {e}")

def generate_personality_analysis(personality_type):
    """
    Generates a personality analysis based on the personality type using AI and translates it to Turkish.
    """
    # Prompt the AI model for analysis
    prompt = f"Give an insightful personality analysis for someone with a '{personality_type}' personality type based on these answers."
    analysis = model.invoke(input=prompt)
    
    # Create an instance of GoogleTranslator
    translator = GoogleTranslator(source='en', target='tr')
    
    # Translate the analysis to Turkish
    translated_analysis = translator.translate(analysis.strip())
    return translated_analysis  # Return the translated analysis

def record_choice(choice):
    """
    Records the user's choice for the current question.
    """
    global user_choices
    question = personality_quiz_data[current_question]

    # Append the selected option text to user choices for tracking
    selected_option = question["options"][choice]  # Update to use "options"
    user_choices.append(selected_option)

    # Disable buttons and enable Next
    for button in choice_btns:
        button.config(state="disabled")
    next_btn.config(state="normal")

def next_question():
    """
    Moves to the next question or calculates the result if the quiz is finished.
    """
    global current_question
    current_question += 1

    # Check if there are remaining questions
    if current_question < len(personality_quiz_data):
        show_question()
    else:
        # Ensure all 10 questions are answered before calculating the result
        if len(user_choices) == len(personality_quiz_data):
            calculate_result()
        else:
            feedback_label.config(
                text="Lütfen tüm soruları cevapladığınızdan emin olun.", 
                foreground="red"
            )


def calculate_result():
    """
    Calculates the most common personality type and displays the result in Turkish.
    """
    personality_type = Counter(user_choices).most_common(1)[0][0]

    # Generate AI-based analysis in Turkish
    analysis = generate_personality_analysis(personality_type)

    # Display the personality type and analysis
    messagebox.showinfo(
        "Kişilik Analiziniz",
        f"Belirgin kişilik tipiniz: {personality_type}.\n\nAnaliz:\n{analysis}"
    )
    root.destroy()

# Main GUI application
root = tk.Tk()
root.title("Kisilik Analizi Uygulamasi")
root.geometry("600x800")
style = Style(theme="flatly")

# Configure styles
style.configure("TLabel", font=("Helvetica", 20))
style.configure("TButton", font=("Helvetica", 16))

# Question label
qs_label = ttk.Label(root, anchor="center", wraplength=500, padding=10)
qs_label.pack(pady=10)

image_label = ttk.Label(root, anchor="center", padding=10)
image_label.pack(pady=10)

# Choice buttons
choice_btns = []
for i in range(4):
    button = ttk.Button(root, command=lambda i=i: record_choice(i))
    button.pack(pady=5)
    choice_btns.append(button)

# Feedback label
feedback_label = ttk.Label(root, anchor="center", padding=10)
feedback_label.pack(pady=10)

# Next button
next_btn = ttk.Button(root, text="Sonraki", command=next_question, state="disabled")
next_btn.pack(pady=10)

# Initialize quiz state
current_question = 0
user_choices = []

# Show the first question
show_question()

# Run the app
root.mainloop()
