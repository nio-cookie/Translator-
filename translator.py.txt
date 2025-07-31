import requests
import json
import base64
import time # Exponential backoff အတွက်

# သင့်ရဲ့ Gemini API Key ကို ဒီနေရာမှာ ထည့်သွင်းပါ။
# လုံခြုံရေးအရ API Key ကို code ထဲမှာ တိုက်ရိုက်ထည့်သွင်းခြင်းသည် အကောင်းဆုံးနည်းလမ်းမဟုတ်ပါ။
# Production environment များအတွက် environment variables သို့မဟုတ် secret management services များကို အသုံးပြုသင့်ပါသည်။
API_KEY = "AIzaSyCZuSjUlhVlDzTg9d5a3GeDoTv3fGb0-Ho"

# Gemini API endpoint များ
TRANSLATION_API_URL = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key={API_KEY}"
TTS_API_URL = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent?key={API_KEY}"

def translate_text(text, source_lang, target_lang):
    """
    စာသားကို ဘာသာပြန်ပေးသည်။
    """
    print(f"\nဘာသာပြန်နေသည်: '{text}' ({source_lang} -> {target_lang})...")
    prompt = ""
    if source_lang == "my" and target_lang == "en":
        prompt = f"Translate the following Burmese text to English, accurately translating any slang or informal language, providing only the translated text: \"{text}\""
    elif source_lang == "en" and target_lang == "my":
        prompt = f"Translate the following English text to Burmese, accurately translating any slang or informal language, providing only the translated text: \"{text}\""
    else:
        return "မပံ့ပိုးသော ဘာသာပြန်လမ်းကြောင်း။", None

    chat_history = [{"role": "user", "parts": [{"text": prompt}]}]
    payload = {"contents": chat_history}

    retries = 0
    max_retries = 3
    base_delay = 0.5 # စက္ကန့်

    while retries < max_retries:
        try:
            response = requests.post(TRANSLATION_API_URL, headers={"Content-Type": "application/json"}, json=payload)
            response.raise_for_status() # HTTP errors များကို စစ်ဆေးပါ (4xx or 5xx)
            result = response.json()

            if result.get("candidates") and len(result["candidates"]) > 0 and \
               result["candidates"][0].get("content") and result["candidates"][0]["content"].get("parts") and \
               len(result["candidates"][0]["content"]["parts"]) > 0:
                translated_text = result["candidates"][0]["content"]["parts"][0]["text"]
                return translated_text, None
            else:
                return "ဘာသာပြန်ခြင်း မအောင်မြင်ပါ။ မမျှော်လင့်သော API တုံ့ပြန်မှု ဖွဲ့စည်းပုံ။", result

        except requests.exceptions.RequestException as e:
            if response.status_code == 429 and retries < max_retries - 1:
                delay = base_delay * (2 ** retries)
                print(f"Rate limit ကျော်လွန်ပါသည်။ {delay:.1f} စက္ကန့်အတွင်း ပြန်ကြိုးစားပါမည်...")
                time.sleep(delay)
                retries += 1
            else:
                return f"ဘာသာပြန်နေစဉ် အမှားတစ်ခု ဖြစ်ပွားခဲ့သည်။: {e}", None
        except json.JSONDecodeError as e:
            return f"API တုံ့ပြန်မှုကို parse လုပ်ရာတွင် အမှားဖြစ်ပွားခဲ့သည်။: {e}", None
    return "ဘာသာပြန်ခြင်း မအောင်မြင်ပါ။ အကြိမ်ကြိမ်ကြိုးစားပြီးနောက်။", None

def speak_text(text, lang_code):
    """
    စာသားကို အသံထွက်ပေးသည်။
    """
    print(f"\nအသံထွက်နေသည်: '{text}' ({lang_code})...")
    voice_name = "Puck" if lang_code == "my-MM" else "Kore" # မြန်မာအတွက် Puck, အင်္ဂလိပ်အတွက် Kore

    payload = {
        "contents": [{"parts": [{"text": text}]}],
        "generationConfig": {
            "responseModalities": ["AUDIO"],
            "speechConfig": {
                "voiceConfig": {"prebuiltVoiceConfig": {"voiceName": voice_name}},
                "languageCode": lang_code
            }
        },
        "model": "gemini-2.5-flash-preview-tts"
    }

    retries = 0
    max_retries = 3
    base_delay = 0.5 # စက္ကန့်

    while retries < max_retries:
        try:
            response = requests.post(TTS_API_URL, headers={"Content-Type": "application/json"}, json=payload)
            response.raise_for_status()
            result = response.json()

            part = result.get("candidates", [{}])[0].get("content", {}).get("parts", [{}])[0]
            audio_data_base64 = part.get("inlineData", {}).get("data")
            mime_type = part.get("inlineData", {}).get("mimeType")

            if audio_data_base64 and mime_type and mime_type.startswith("audio/L16"):
                # base64 encoded audio data ကို ပြန်ဖြည်ပါ။
                audio_bytes = base64.b64decode(audio_data_base64)

                # WAV header ကို ကိုယ်တိုင်ထည့်သွင်းပါ။
                sample_rate_match = next((int(s.split('=')[1]) for s in mime_type.split(';') if 'rate=' in s), 16000)
                num_channels = 1
                bytes_per_sample = 2 # 16-bit PCM
                block_align = num_channels * bytes_per_sample
                byte_rate = sample_rate_match * block_align
                data_length = len(audio_bytes)

                wav_header = b'RIFF'
                wav_header += (36 + data_length).to_bytes(4, 'little')
                wav_header += b'WAVE'
                wav_header += b'fmt '
                wav_header += (16).to_bytes(4, 'little') # Subchunk1Size (16 for PCM)
                wav_header += (1).to_bytes(2, 'little') # AudioFormat (1 for PCM)
                wav_header += num_channels.to_bytes(2, 'little')
                wav_header += sample_rate_match.to_bytes(4, 'little')
                wav_header += byte_rate.to_bytes(4, 'little')
                wav_header += block_align.to_bytes(2, 'little')
                wav_header += (bytes_per_sample * 8).to_bytes(2, 'little') # BitsPerSample
                wav_header += b'data'
                wav_header += data_length.to_bytes(4, 'little')

                full_audio_data = wav_header + audio_bytes

                # အသံဖိုင်ကို သိမ်းဆည်းပါ။
                output_filename = "translated_audio.wav"
                with open(output_filename, "wb") as f:
                    f.write(full_audio_data)
                print(f"အသံဖိုင်ကို '{output_filename}' အဖြစ် သိမ်းဆည်းပြီးပါပြီ။")
                print("ဒီဖိုင်ကို သင့်ကွန်ပျူတာပေါ်မှာ ဖွင့်နိုင်ပါတယ်။")
                return True
            else:
                print("TTS: မမျှော်လင့်သော တုံ့ပြန်မှု ဖွဲ့စည်းပုံ သို့မဟုတ် audio data မရှိပါ။")
                return False

        except requests.exceptions.RequestException as e:
            if response.status_code == 429 and retries < max_retries - 1:
                delay = base_delay * (2 ** retries)
                print(f"Rate limit ကျော်လွန်ပါသည်။ {delay:.1f} စက္ကန့်အတွင်း ပြန်ကြိုးစားပါမည်...")
                time.sleep(delay)
                retries += 1
            else:
                print(f"TTS ခေါ်ဆိုရာတွင် အမှားတစ်ခု ဖြစ်ပွားခဲ့သည်။: {e}")
                return False
        except json.JSONDecodeError as e:
            print(f"API တုံ့ပြန်မှုကို parse လုပ်ရာတွင် အမှားဖြစ်ပွားခဲ့သည်။: {e}")
            return False
    print("TTS မအောင်မြင်ပါ။ အကြိမ်ကြိမ်ကြိုးစားပြီးနောက်။")
    return False

if __name__ == "__main__":
    print("Gemini Translator (Python Console Version)")
    print("----------------------------------------")

    while True:
        print("\nဘာသာပြန်လမ်းကြောင်း ရွေးချယ်ပါ:")
        print("1. မြန်မာ -> အင်္ဂလိပ်")
        print("2. အင်္ဂလိပ် -> မြန်မာ")
        print("3. ထွက်ရန်")

        choice = input("သင့်ရွေးချယ်မှု (1/2/3): ")

        if choice == '1':
            source_lang = "my"
            target_lang = "en"
            user_input = input("ဘာသာပြန်လိုသော မြန်မာစာသားကို ရိုက်ထည့်ပါ: ")
        elif choice == '2':
            source_lang = "en"
            target_lang = "my"
            user_input = input("ဘာသာပြန်လိုသော အင်္ဂလိပ်စာသားကို ရိုက်ထည့်ပါ: ")
        elif choice == '3':
            print("ပရိုဂရမ်မှ ထွက်ခွာပါပြီ။")
            break
        else:
            print("မှားယွင်းသော ရွေးချယ်မှု။ ကျေးဇူးပြု၍ 1, 2, သို့မဟုတ် 3 ကို ရိုက်ပါ။")
            continue

        if user_input.strip() == "":
            print("ဘာသာပြန်ရန် စာသားထည့်ပါ။")
            continue

        translated_text, error_info = translate_text(user_input, source_lang, target_lang)

        if translated_text:
            print(f"\nဘာသာပြန်ထားသော စာသား: {translated_text}")
            if error_info:
                print(f"API တုံ့ပြန်မှု အပြည့်အစုံ: {json.dumps(error_info, indent=2)}")

            speak_choice = input("ဘာသာပြန်ထားသော စာသားကို အသံထွက်လိုပါသလား (y/n)? ").lower()
            if speak_choice == 'y':
                # TTS အတွက် language code ကို သတ်မှတ်ပါ။
                tts_lang_code = "en-US" if target_lang == "en" else "my-MM"
                speak_text(translated_text, tts_lang_code)
        else:
            print(f"\nဘာသာပြန်ခြင်း မအောင်မြင်ပါ။: {translated_text}")
            if error_info:
                print(f"API တုံ့ပြန်မှု အပြည့်အစုံ: {json.dumps(error_info, indent=2)}")


