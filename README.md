   import requests
import random
import datetime
from collections import Counter

# --- 1. FETCH ACTUAL NYT DATA ---

def get_todays_wordle_answer():
    """
    Fetches the official solution for TODAY from the NYT API.
    """
    # Get current date in YYYY-MM-DD format
    today = datetime.date.today().strftime("%Y-%m-%d")
    
    # NYT Endpoint for daily Wordle data
   url = f"https://www.nytimes.com/svc/wordle/v2/{today}.json"
    
   try:
        # We need a User-Agent so NYT doesn't block the script
        headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
        }
        response = requests.get(url, headers=headers)
             if response.status_code == 200:
            data = response.json()
            solution = data.get("solution")
            days_since_launch = data.get("days_since_launch")
            print(f"✅ Successfully fetched NYT Wordle #{days_since_launch} for {today}")
            return solution.lower()
        else:
            print(f"⚠️ Could not fetch data from NYT (Status: {response.status_code}).")
            return None
                except Exception as e:
        print(f"⚠️ Connection error: {e}")
        return None

# --- 2. SOLVER ENGINE (Logic to play the game) ---

def load_word_list():
    """Fetches a dictionary so the bot has a vocabulary to make guesses with."""
    url = "https://raw.githubusercontent.com/tabatkins/wordle-list/main/words"
    try:
        response = requests.get(url)
        words = response.text.splitlines()
        return [w.lower() for w in words if len(w) == 5]
    except:
        return []

def score_words(words, letter_counts):
    """Sorts words by how much information they reveal (based on letter freq)."""
    def get_word_score(word):
        unique_chars = set(word)
        return sum(letter_counts[char] for char in unique_chars)
    return sorted(words, key=get_word_score, reverse=True)

def generate_feedback(target, guess):
    """Simulates the Green/Yellow/Gray response."""
    result = list("BBBBB") 
    target_chars = list(target)
        # Pass 1: Green (Correct pos)
    for i in range(5):
        if guess[i] == target[i]:
            result[i] = 'G'
            target_chars[i] = None 
                # Pass 2: Yellow (Wrong pos)
    for i in range(5):
        if result[i] == 'B': 
            if guess[i] in target_chars:
                result[i] = 'Y'
                target_chars[target_chars.index(guess[i])] = None
                    return "".join(result)

def filter_words(words, guess, result):
    """Removes words that don't match the feedback."""
    new_candidates = []
    for word in words:
        match = True
        for i, (char_guess, color) in enumerate(zip(guess, result)):
            if color == 'G':
                if word[i] != char_guess:
                    match = False; break
            elif color == 'Y':
                if char_guess not in word or word[i] == char_guess:
                    match = False; break
            elif color == 'B':
                if char_guess in word:
                    # If this letter is in the word but marked black here:
                    # 1. It must not be at this specific index (word[i] != char)
                    # 2. If we haven't seen it as G/Y elsewhere in this guess, it shouldn't be in the word at all.
                    if word[i] == char_guess:
                        match = False; break
                                   is_duplicate_in_guess = guess.count(char_guess) > 1
                    if not is_duplicate_in_guess:
                        match = False; break
        if match:
            new_candidates.append(word)
    return new_candidates

# --- 3. MAIN AUTOMATION ---

def main():
    print("--- 🌍 REAL-TIME NYT WORDLE SOLVER 🌍 ---")
    
  
  target_word = get_todays_wordle_answer()
    
  if not target_word:
        print("Using random fallback word due to connection failure.")
        target_word = "error" 


  candidates = load_word_list()
    

   if target_word not in candidates:
        candidates.append(target_word)
    global_letter_counts = Counter("".join(candidates))
    current_guess = "slate" # Proven strong starter
        print(f"\n🎯 TARGET LOCKED: {target_word.upper()} (Hidden from bot logic)")
    print("-" * 30)
    for attempt in range(1, 7):
        print(f"Attempt {attempt}: {current_guess.upper()}")
                if current_guess == target_word:
            print(f"\n✨ SOLVED! The word for today is {target_word.upper()}.")
            print(f"Stats: {attempt}/6 guesses.")
            break
                 feedback = generate_feedback(target_word, current_guess)
                  print(f"Feedback : {feedback}  (G=Green, Y=Yellow, B=Gray)")
            candidates = filter_words(candidates, current_guess, feedback)
        print(f"Words left: {len(candidates)}")
          if not candidates:
            print("Error: No words match the feedback.")
            break
            
  # Pick next best word
   if len(candidates) == 1:
            current_guess = candidates[0]
   else:
            current_counts = Counter("".join(candidates))
            current_guess = score_words(candidates, current_counts)[0]

if __name__ == "__main__":
    main()
