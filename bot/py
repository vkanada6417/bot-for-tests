import telebot
import json
from PIL import Image
from io import BytesIO
import base64
import time
import requests
import os
import google.generativeai as genai

# Bot setup
BOT_TOKEN = ''
bot = telebot.TeleBot(BOT_TOKEN)

# FusionBrain API
class FusionBrainAPI:
    def __init__(self, url, api_key, secret_key):
        self.URL = url
        self.AUTH_HEADERS = {
            'X-Key': f'Key {api_key}',
            'X-Secret': f'Secret {secret_key}',
        }

    def get_pipeline(self):
        response = requests.get(self.URL + 'key/api/v1/pipelines', headers=self.AUTH_HEADERS)
        data = response.json()
        return data[0]['id']

    def generate(self, prompt, pipeline, images=1, width=1024, height=1024):
        params = {
            "type": "GENERATE",
            "numImages": images,
            "width": width,
            "height": height,
            "generateParams": {
                "query": f"{prompt}"
            }
        }
        data = {
            'pipeline_id': (None, pipeline),
            'params': (None, json.dumps(params), 'application/json')
        }
        response = requests.post(self.URL + 'key/api/v1/pipeline/run', headers=self.AUTH_HEADERS, files=data)
        data = response.json()
        return data['uuid']

    def check_generation(self, request_id, attempts=10, delay=10):
        while attempts > 0:
            response = requests.get(self.URL + 'key/api/v1/pipeline/status/' + request_id, headers=self.AUTH_HEADERS)
            data = response.json()
            if data['status'] == 'DONE':
                return data['result']['files']
            attempts -= 1
            time.sleep(delay)
        raise TimeoutError("Generation timed out. Please try again.")

    def save_image_from_base64(self, base64_string):
        try:
            decoded_data = base64.b64decode(base64_string)
            image = Image.open(BytesIO(decoded_data))
            return image
        except Exception as e:
            print(f"Error processing image: {e}")
            return None

fusion_brain_api = FusionBrainAPI(
    url='https://api-key.fusionbrain.ai/',
    api_key='',
    secret_key='A'
)

# Gemini setup
GOOGLE_API_KEY = ""  # Your Gemini API key
genai.configure(api_key=GOOGLE_API_KEY)

def chat_with_gemini(user_input):
    """
    Uses Gemini 2.0 Flash to respond to user queries or refine prompts.
    """
    model = genai.GenerativeModel('models/gemini-2.0-flash')
    response = model.generate_content(user_input)
    return response.text.strip()

# Helper function to create inline keyboard
def get_options_keyboard():
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("Generate Image (FusionBrain)", callback_data="fusionbrain"),
        telebot.types.InlineKeyboardButton("Chat with Gemini", callback_data="gemini")
    )
    return markup

# Start command handler
@bot.message_handler(commands=['start', 'help'])
def send_welcome(message):
    bot.reply_to(message, "Hello! I'm a bot that can generate images or chat with you using Gemini. "
                          "Choose an option below:", reply_markup=get_options_keyboard())

# Callback query handler
@bot.callback_query_handler(func=lambda call: True)
def handle_callback(call):
    chat_id = call.message.chat.id
    if call.data == "fusionbrain":
        bot.send_message(chat_id, "Please describe the image you want to generate.")
        bot.register_next_step_handler(call.message, generate_image_with_fusionbrain)
    elif call.data == "gemini":
        bot.send_message(chat_id, "Send me a message, and I'll respond using Gemini.")
        bot.register_next_step_handler(call.message, chat_with_gemini_handler)

# Generate image with FusionBrain
def generate_image_with_fusionbrain(message):
    user_prompt = message.text
    generating_message = bot.send_message(message.chat.id, "Generating image...")
    bot.send_chat_action(message.chat.id, 'typing')

    try:
        pipeline_id = fusion_brain_api.get_pipeline()
        uuid = fusion_brain_api.generate(user_prompt, pipeline_id)
        files = fusion_brain_api.check_generation(uuid)[0]
        image = fusion_brain_api.save_image_from_base64(files)

        if image:
            temp_file_path = "temp_image.jpg"
            image.save(temp_file_path, format="JPEG")
            with open(temp_file_path, 'rb') as photo:
                bot.send_photo(message.chat.id, photo=photo)
            bot.delete_message(message.chat.id, generating_message.message_id)
            os.remove(temp_file_path)
        else:
            bot.send_message(message.chat.id, "Sorry, I couldn't generate the image.")
    except Exception as e:
        bot.send_message(message.chat.id, f"An error occurred: {str(e)}")

# Chat with Gemini
def chat_with_gemini_handler(message):
    user_input = message.text
    try:
        response = chat_with_gemini(user_input)
        bot.send_message(message.chat.id, f"Gemini: {response}")
    except Exception as e:
        bot.send_message(message.chat.id, f"An error occurred: {str(e)}")

if __name__ == '__main__':
    bot.polling(none_stop=True)
