[dungeon_cli.py](https://github.com/user-attachments/files/29135758/dungeon_cli.py)

#!/usr/bin/env python3
"""
DUNGEON OF FATE - Enhanced Edition
A proper old-school D&D-inspired solo text RPG with free-form commands.

You can type almost anything like a real player at the table:
  "roll d20"
  "roll 2d6+3"
  "venture deeper"
  "search"
  "cast magic missile"
  "help combat"
  "equip longsword"
  "stats"
  "map"

New features:
- Choose a class (Warrior / Rogue / Mage) with unique abilities
- Real inventory + equippable weapons and armor
- Mage spells with limited uses
- Proper dice parser (supports standard D&D notation)
- ASCII dungeon map view
- Extensive in-game help and tutorial for rusty or new players
- Class-specific actions and better loot

Run:
    python dungeon_cli.py

Pure Python. No dependencies besides the dice_utils.py in the same folder.
"""

import random
import time
import sys
import re
import os
import json
from typing import List, Dict, Optional, Tuple

try:
    from dice_utils import roll, parse_dice_expression, roll_from_text, advantage_disadvantage
except ImportError:
    print("ERROR: dice_utils.py must be in the same folder as dungeon_cli.py")
    sys.exit(1)

# ====================== OPTIONAL TTS (Text-to-Speech) ======================
# For dyslexic players: type "speak on" to enable narration.
# Requires: pip install pyttsx3
TTS_ENABLED = False
TTS_ENGINE = None

def init_tts():
    global TTS_ENGINE
    try:
        import pyttsx3
        TTS_ENGINE = pyttsx3.init()
        # On Windows this uses the system voice (usually very good)
        # You can tweak rate if you want:
        # TTS_ENGINE.setProperty('rate', 185)
        return True
    except Exception as e:
        print("TTS not available (optional). To enable voice narration for dyslexic players:")
        print("    pip install pyttsx3")
        print("Then restart the game and type 'speak on'.")
        return False

def speak(text):
    """Speak text if TTS is enabled and working."""
    global TTS_ENABLED
    if not TTS_ENABLED or not TTS_ENGINE or not text:
        return
    try:
        clean = str(text).replace("\n", " ").strip()
        if clean:
            TTS_ENGINE.say(clean)
            TTS_ENGINE.runAndWait()
    except Exception:
        pass  # Silently fail so it never breaks the game

def toggle_tts(enable: bool):
    global TTS_ENABLED
    if enable:
        if TTS_ENGINE is None:
            if not init_tts():
                return False
        TTS_ENABLED = True
        print("🔊 Voice narration enabled. The game will read key descriptions aloud.")
        speak("Voice narration enabled.")
        return True
    else:
        TTS_ENABLED = False
        print("Voice narration disabled.")
        return True

# ====================== SAVE / LOAD SYSTEM ======================
SAVES_DIR = "saves"

def ensure_saves_dir():
    """Create the saves folder if it doesn't exist."""
    if not os.path.exists(SAVES_DIR):
        try:
            os.makedirs(SAVES_DIR)
        except Exception:
            pass

def get_save_path(slot: str) -> str:
    safe_name = re.sub(r'[^a-zA-Z0-9_-]', '_', slot)[:40]
    return os.path.join(SAVES_DIR, f"{safe_name}.json")

def save_game(player: Dict, slot: Optional[str] = None) -> bool:
    """Save the current player state to a JSON file."""
    ensure_saves_dir()
    if not slot:
        slot = f"{player.get('name', 'Adventurer')}_{player.get('depth', 0)}"
    path = get_save_path(slot)
    try:
        save_data = {
            "version": 1,
            "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
            "player": player,
        }
        with open(path, "w", encoding="utf-8") as f:
            json.dump(save_data, f, indent=2, ensure_ascii=False)
        print(f"✅ Game saved to '{slot}' ({path})")
        speak(f"Game saved as {slot}.")
        return True
    except Exception as e:
        print(f"Failed to save game: {e}")
        return False

def load_game(slot: str) -> Optional[Dict]:
    """Load a player state from a save file."""
    path = get_save_path(slot)
    if not os.path.exists(path):
        print(f"No save found for '{slot}'.")
        return None
    try:
        with open(path, "r", encoding="utf-8") as f:
            data = json.load(f)
        player = data.get("player")
        if player:
            print(f"✅ Loaded save '{slot}' from {data.get('timestamp', 'unknown time')}")
            speak("Save loaded. Welcome back, adventurer.")
            return player
    except Exception as e:
        print(f"Failed to load save: {e}")
    return None

def list_saves() -> List[str]:
    """Return a list of available save slot names."""
    ensure_saves_dir()
    saves = []
    try:
        for fname in os.listdir(SAVES_DIR):
            if fname.endswith(".json"):
                saves.append(fname[:-5])  # remove .json
    except Exception:
        pass
    return sorted(saves)

def delete_save(slot: str) -> bool:
    path = get_save_path(slot)
    if os.path.exists(path):
        try:
            os.remove(path)
            print(f"Deleted save '{slot}'.")
            return True
        except Exception as e:
            print(f"Could not delete: {e}")
    else:
        print("No such save.")
    return False


# ====================== DATA: CLASSES, ITEMS, SPELLS, MONSTERS ======================

CLASS_DATA = {
    "warrior": {
        "name": "Warrior",
        "hp_bonus": 4,
        "attack_bonus": 1,
        "defense_bonus": 1,
        "description": "Tough and reliable. Bonus HP and better defense. Can do a POWER ATTACK once per fight for extra damage.",
        "starting_gear": ["Iron Longsword", "Chain Shirt"],
        "special": "power attack",
        "proficient": ["heavy", "medium", "light", "staff"]
    },
    "rogue": {
        "name": "Rogue",
        "hp_bonus": 0,
        "attack_bonus": 2,
        "defense_bonus": 0,
        "description": "Fast and deadly. Better chance to hit and can SNEAK ATTACK for bonus damage at the start of combat or on a good roll.",
        "starting_gear": ["Dagger of Opportunity", "Leather Armor"],
        "special": "sneak attack",
        "proficient": ["medium", "light"]
    },
    "mage": {
        "name": "Mage",
        "hp_bonus": -3,
        "attack_bonus": 0,
        "defense_bonus": 0,
        "description": "Fragile but powerful. Knows several spells with limited uses each run. Can CAST in combat instead of a normal attack.",
        "starting_gear": ["Staff of the Apprentice", "Robes"],
        "special": "spells",
        "proficient": ["light", "staff"]
    }
}

SPELLS = {
    "magic missile": {
        "name": "Magic Missile",
        "uses": 3,
        "desc": "Auto-hit for 1d4+1 force damage. Reliable.",
        "effect": lambda p: (roll(4) + 1, "force")
    },
    "burning hands": {
        "name": "Burning Hands",
        "uses": 2,
        "desc": "2d6 fire damage, but you must succeed a d20 roll (DC 12) or take some backlash.",
        "effect": lambda p: (roll(6) + roll(6), "fire")
    },
    "shield": {
        "name": "Shield",
        "uses": 2,
        "desc": "Gain +4 to defense for the rest of this combat. (One time use per casting)",
        "effect": None   # special handling
    }
}

ITEM_TEMPLATES = [
    {"name": "Rusty Dagger", "type": "weapon", "attack_mod": 0, "value": 3, "category": "light"},
    {"name": "Iron Longsword", "type": "weapon", "attack_mod": 1, "value": 12, "category": "medium"},
    {"name": "Fine Rapier", "type": "weapon", "attack_mod": 2, "value": 25, "category": "light"},
    {"name": "Orcish Greataxe", "type": "weapon", "attack_mod": 3, "value": 30, "category": "heavy"},
    {"name": "Leather Armor", "type": "armor", "defense_mod": 1, "value": 10, "category": "light"},
    {"name": "Chain Shirt", "type": "armor", "defense_mod": 2, "value": 22, "category": "medium"},
    {"name": "Breastplate", "type": "armor", "defense_mod": 3, "value": 40, "category": "heavy"},
    {"name": "Potion of Healing", "type": "consumable", "heal": 8, "value": 8},
    {"name": "Greater Healing Potion", "type": "consumable", "heal": 14, "value": 18},
    {"name": "Scroll of Fire", "type": "consumable", "damage": 10, "value": 15},
    {"name": "Gygax's Guiding Die", "type": "trinket", "value": 60, "desc": "A twenty-sided die etched with tiny dungeon rooms instead of numbers. It whispers the wisdom of the Grandmaster."},
    {"name": "Staff of the Apprentice", "type": "weapon", "attack_mod": 0, "value": 5, "category": "staff"},
]

MONSTER_TABLE = [
    {"name": "Goblin Scavenger", "hp": 7,  "ac": 11, "dmg": 3, "xp": 8,  "desc": "a snickering little green menace with a rusty knife"},
    {"name": "Skeletal Warrior", "hp": 10, "ac": 13, "dmg": 4, "xp": 12, "desc": "bones clacking as it raises a notched sword"},
    {"name": "Cave Bat Swarm",   "hp": 6,  "ac": 14, "dmg": 2, "xp": 6,  "desc": "a cloud of screeching wings and teeth"},
    {"name": "Orc Brute",        "hp": 14, "ac": 12, "dmg": 6, "xp": 18, "desc": "a hulking brute with a blood-stained axe"},
    {"name": "Shadow Stalker",   "hp": 9,  "ac": 15, "dmg": 5, "xp": 15, "desc": "something half-seen that whispers your name"},
    {"name": "Carrion Crawler",  "hp": 12, "ac": 13, "dmg": 5, "xp": 14, "desc": "a many-legged horror with paralyzing tendrils"},
]

BOSS = {
    "name": "The Crypt Warden",
    "hp": 30,
    "ac": 16,
    "dmg": 7,
    "xp": 55,
    "desc": "an ancient armored guardian whose eyes glow with cold blue fire"
}

ROOM_FLAVORS = [
    "The air is thick with the smell of wet stone and old blood.",
    "Flickering torches cast long dancing shadows on the walls.",
    "You hear distant dripping water... and something breathing.",
    "Moss and strange glowing fungi light the passage in sickly green.",
    "Carvings of forgotten gods watch you from the cracked walls.",
    "A cold wind blows from somewhere deeper. It almost sounds like laughter.",
    "Broken statues lie toppled. One still clutches a real weapon.",
    "The floor is slick with something you hope is water.",
]

LOOT_DESCRIPTIONS = [
    "You pry open a warped chest.",
    "A skeleton's bony fingers still grip a small satchel.",
    "You spot a glint inside a cracked wall niche.",
    "Kicking over rubble reveals something useful.",
]


# ====================== PLAYER & INVENTORY ======================

def create_character() -> Dict:
    print("\n" + "=" * 56)
    print("          WELCOME TO THE DUNGEON OF FATE")
    print("              (Enhanced Old-School Edition)")
    print("=" * 56)

    print("\nIt's been years? No worries. This is a solo D&D-style game.")
    print("Everything is decided by dice. You can type commands like a real player.")
    print("At any time type 'help' or 'tutorial' if you get lost.")
    print("For dyslexic support: type 'speak on' after starting (needs 'pip install pyttsx3').\n")

    name = input("What is your adventurer's name? ").strip()
    if not name:
        name = "Nameless Wanderer"

    print(f"\n{name}... the dice decide your path.")
    time.sleep(0.6)

    # Class choice
    print("\nChoose your class (type the name or number):")
    print("  1. Warrior  - Tough, reliable, POWER ATTACK")
    print("  2. Rogue    - Sneaky, high accuracy, SNEAK ATTACK")
    print("  3. Mage     - Fragile spellcaster with MAGIC MISSILE and more")
    while True:
        choice = input("> ").strip().lower()
        if choice in ("1", "warrior", "w"):
            cls = "warrior"
            break
        elif choice in ("2", "rogue", "r"):
            cls = "rogue"
            break
        elif choice in ("3", "mage", "m", "wizard", "magic"):
            cls = "mage"
            break
        else:
            print("Please choose 1 (Warrior), 2 (Rogue), or 3 (Mage).")

    cdata = CLASS_DATA[cls]
    print(f"\nYou are a {cdata['name']}.")

    # Base rolls
    base_hp = 16 + roll(8)
    hp = max(8, base_hp + cdata["hp_bonus"])
    attack_bonus = roll(6) // 2 + cdata["attack_bonus"]
    defense_bonus = cdata["defense_bonus"]

    # Difficulty selection for new progression system
    print("\nChoose your challenge:")
    print("  1. Easy       - 8 depths (classic)")
    print("  2. Hard       - 15 depths")
    print("  3. You're Insane - 30 depths (Drakulich awaits...)")
    while True:
        diff_choice = input("> ").strip()
        if diff_choice in ("1", "easy"):
            difficulty = "easy"
            max_depth = 8
            break
        elif diff_choice in ("2", "hard"):
            difficulty = "hard"
            max_depth = 15
            break
        elif diff_choice in ("3", "insane", "you're insane"):
            difficulty = "insane"
            max_depth = 30
            break
        else:
            print("Please choose 1 (Easy), 2 (Hard), or 3 (You're Insane).")

    player = {
        "name": name,
        "class": cls,
        "class_name": cdata["name"],
        "hp": hp,
        "max_hp": hp,
        "attack_bonus": attack_bonus,
        "defense_bonus": defense_bonus,
        "gold": 10,
        "xp": 0,
        "level": 1,
        "xp_to_next": 100,
        "kills": 0,
        "depth": 0,
        "difficulty": difficulty,
        "max_depth": max_depth,
        "proficient": cdata.get("proficient", []),
        "inventory": [],
        "equipped": {"weapon": None, "armor": None, "trinket": None},
        "spell_uses": {},   # for mage
        "special_used": False,  # per combat reset
    }

    # Starting gear
    for item_name in cdata["starting_gear"]:
        item = _make_item(item_name)
        if item:
            player["inventory"].append(item)
            if item["type"] == "weapon" and not player["equipped"]["weapon"]:
                player["equipped"]["weapon"] = item
            if item["type"] == "armor" and not player["equipped"]["armor"]:
                player["equipped"]["armor"] = item

    if cls == "mage":
        for spell_id in SPELLS:
            player["spell_uses"][spell_id] = SPELLS[spell_id]["uses"]

    print(f"\n  Class         : {cdata['name']}")
    print(f"  Hit Points    : {player['hp']}")
    print(f"  Attack Bonus  : +{player['attack_bonus']}")
    print(f"  Defense Bonus : +{player['defense_bonus']}")
    print(f"  Difficulty    : {difficulty.title()} ({max_depth} depths)")
    print(f"  Starting gear : {', '.join(cdata['starting_gear'])}")
    print(f"\n{cdata['description']}")
    print("\nPress Enter to descend into the crypt...")
    input()
    return player


def _make_item(template_name: str) -> Optional[Dict]:
    for t in ITEM_TEMPLATES:
        if t["name"].lower() == template_name.lower():
            item = t.copy()
            item["id"] = item["name"].lower().replace(" ", "_")
            return item
    # Fallback random
    t = random.choice(ITEM_TEMPLATES)
    item = t.copy()
    item["id"] = item["name"].lower().replace(" ", "_")
    return item


def get_attack_bonus(player: Dict) -> int:
    mod = player["attack_bonus"]
    w = player["equipped"].get("weapon")
    if w:
        if w.get("category") and player.get("proficient") and w["category"] not in player["proficient"]:
            mod -= 2  # penalty for non-proficient gear
        if "attack_mod" in w:
            mod += w["attack_mod"]
    t = player["equipped"].get("trinket")
    if t and t.get("name") == "Gygax's Guiding Die":
        mod += 1  # The master's wisdom guides your strikes
    return mod


def get_defense(player: Dict) -> int:
    base = 12 + player["defense_bonus"]
    a = player["equipped"].get("armor")
    if a:
        if a.get("category") and player.get("proficient") and a["category"] not in player["proficient"]:
            base -= 2  # penalty for non-proficient gear
        if "defense_mod" in a:
            base += a["defense_mod"]
    return base

def gain_xp(player: Dict, amount: int):
    """Award XP and check for level ups."""
    player["xp"] += amount
    leveled = False
    while player["xp"] >= player["xp_to_next"]:
        player["level"] += 1
        hp_gain = 4 + (player["level"] // 2)
        player["max_hp"] += hp_gain
        player["hp"] = min(player["max_hp"], player["hp"] + hp_gain // 2)
        if player["level"] % 2 == 0:
            player["attack_bonus"] += 1
        if player["level"] % 3 == 0:
            player["defense_bonus"] += 1
        player["xp_to_next"] = 100 * player["level"]
        print(f"\n*** LEVEL UP! You are now level {player['level']}! (+{hp_gain} max HP) ***")
        leveled = True
    if leveled:
        speak(f"Level up! You are now level {player['level']}.")

def get_level_display(player: Dict) -> str:
    return f"Lvl {player.get('level', 1)} ({player['xp']}/{player['xp_to_next']})"


def show_inventory(player: Dict):
    print("\n=== INVENTORY ===")
    if not player["inventory"]:
        print("You carry nothing but courage and bad decisions.")
        return

    for i, item in enumerate(player["inventory"], 1):
        equipped = ""
        if player["equipped"]["weapon"] and item is player["equipped"]["weapon"]:
            equipped = " (equipped - weapon)"
        elif player["equipped"]["armor"] and item is player["equipped"]["armor"]:
            equipped = " (equipped - armor)"
        val = f" (worth {item.get('value', 0)}g)" if "value" in item else ""
        print(f"  {i}. {item['name']}{equipped}{val}")

    print(f"\nGold: {player['gold']}")


def equip_item(player: Dict, item_name: str) -> bool:
    name_lower = item_name.lower()
    for item in player["inventory"]:
        if name_lower in item["name"].lower() or name_lower in item.get("id", ""):
            if item["type"] == "weapon":
                player["equipped"]["weapon"] = item
                prof = item.get("category", "unknown")
                if item.get("category") and player.get("proficient") and item["category"] not in player["proficient"]:
                    print(f"You wield the {item['name']}. (Not proficient: -2 attack)")
                else:
                    print(f"You wield the {item['name']}.")
                return True
            elif item["type"] == "armor":
                player["equipped"]["armor"] = item
                prof = item.get("category", "unknown")
                if item.get("category") and player.get("proficient") and item["category"] not in player["proficient"]:
                    print(f"You don the {item['name']}. (Not proficient: -2 defense)")
                else:
                    print(f"You don the {item['name']}.")
                return True
            elif item["type"] == "trinket":
                player["equipped"]["trinket"] = item
                print(f"You keep the {item['name']} close. The Grandmaster's wisdom is with you.")
                return True
            else:
                print("That can't be equipped.")
                return False
    print("You don't have anything like that.")
    return False


def use_consumable(player: Dict, item_name: str) -> bool:
    name_lower = item_name.lower()
    for i, item in enumerate(player["inventory"]):
        if name_lower in item["name"].lower():
            if item["type"] == "consumable":
                if "heal" in item:
                    heal = item["heal"] + roll(4)
                    player["hp"] = min(player["max_hp"], player["hp"] + heal)
                    print(f"You drink the {item['name']} and recover {heal} HP!")
                elif "damage" in item:
                    print(f"The scroll flares! But you have no target here.")
                player["inventory"].pop(i)
                return True
            else:
                print("You can't use that right now.")
                return False
    print("You don't have that.")
    return False


# ====================== DICE & COMMAND PARSING ======================

def handle_free_roll(text: str) -> bool:
    """If the text looks like a dice command, roll it and print. Return True if handled."""
    result = parse_dice_expression(text)
    if result:
        total, breakdown = result
        print(f"\n🎲 {breakdown}")
        # Sometimes add a little color commentary
        if total >= 18:
            print("   (The gods are smiling...)")
        elif total <= 3:
            print("   (Oof. The dice hate you today.)")
        return True
    return False


def parse_command(text: str) -> str:
    """Normalize player input into a canonical action token."""
    t = text.lower().strip()

    # Dice commands are handled specially before this
    if any(x in t for x in ["roll", "d20", "d6", "d8", "d4", "d10", "d12", "d%"]):
        return "roll"

    # Common aliases
    if t in ("v", "venture", "deeper", "forward", "go", "explore", "descend"):
        return "venture"
    if t in ("s", "search", "look", "loot", "scavenge"):
        return "search"
    if "potion" in t or t in ("drink", "heal", "use potion"):
        return "potion"
    if t in ("rest", "bind", "sleep", "camp"):
        return "rest"
    if t in ("i", "inv", "inventory", "bag", "gear"):
        return "inventory"
    if "equip" in t or "wield" in t or "wear" in t:
        return "equip"
    if t in ("stats", "sheet", "character", "status", "st"):
        return "stats"
    if t in ("map", "m", "where", "level"):
        return "map"
    if t in ("flee", "run", "escape", "leave", "surface"):
        return "flee"
    if t in ("q", "quit", "exit", "give up"):
        return "quit"
    if "cast" in t or "spell" in t:
        return "cast"
    if "power" in t or "power attack" in t:
        return "power_attack"
    if "sneak" in t:
        return "sneak_attack"
    if t in ("help", "?", "h", "commands", "tutorial"):
        return "help"
    if "attack" in t:
        return "attack"

    return "unknown"


# ====================== ASCII MAP ======================

def show_map(player: Dict):
    depth = player["depth"]
    print("\n=== THE CRYPT (your rough mental map) ===")
    print("Surface")
    for d in range(1, 9):
        marker = "  <-- YOU ARE HERE" if d == depth else ""
        bar = "│" if d < depth else " "
        room = "▓" if d <= depth else "░"
        print(f"{bar} {d:2d} {room}{marker}")
    print("      (deeper darkness below)")
    print("Type 'venture' to go deeper. Depths 7+ are dangerous.")


# ====================== HELP / TUTORIAL ======================

def show_help(topic: str = ""):
    print("\n" + "=" * 56)
    print("                      HOW TO PLAY")
    print("=" * 56)

    if not topic or topic in ("help", "commands", "general"):
        print("""
You are exploring a dangerous crypt. Your goal is to go as deep as you can,
gather gold and XP, and survive.

At the main prompt you can type almost anything:

CORE ACTIONS
  venture / go deeper / explore     → Move to the next depth (risky)
  search / loot                     → Look for treasure or trouble
  rest / bind wounds                → Small heal (sometimes attracts monsters)
  potion / drink potion             → Use a healing potion from inventory

INFORMATION
  stats / sheet                     → Your full character info
  inventory / inv                   → See what you're carrying
  map                               → Show your current depth visually

COMBAT ACTIONS (when fighting)
  attack                            → Normal attack
  cast magic missile                → (Mage only) Use a spell
  power attack                      → (Warrior) Big hit, once per fight
  sneak attack                      → (Rogue) Extra damage at start of fight
  potion                            → Drink in the middle of battle
  flee                              → Risky escape roll

FREE DICE ROLLING (any time!)
  roll d20
  roll 2d6+3
  d8
  3d6
  d%

EQUIPMENT
  equip longsword
  wield rapier
  wear chain shirt

OTHER
  help combat
  help mage / help spells
  help classes
  quit

The game is deliberately old-school and random. Good luck!
""")

    if topic in ("combat", "fight", "attack"):
        print("""
COMBAT BASICS
- You and the monster take turns.
- You roll d20 + your attack bonus. If you meet or beat the monster's AC, you hit.
- Damage is usually d6 + your strength/gear bonus.
- Natural 20 (before bonus) is a critical hit = double damage.
- Your effective defense = 12 + defense bonus + armor.
- You can type 'roll d20' in combat if you want to see the dice yourself.
- Mages can 'cast magic missile' etc instead of a normal swing.
""")

    if topic in ("mage", "spells", "cast"):
        print("""
MAGE SPELLS
You start with limited uses of:
  Magic Missile   (reliable auto-hit damage)
  Burning Hands   (big damage but risky)
  Shield          (temporary big defense boost)

In combat just type: cast magic missile
Spells have limited charges for the whole run — use them wisely.
Mages have fewer hit points but can end fights fast.
""")

    if topic in ("classes", "class", "warrior", "rogue"):
        print("""
CLASSES
WARRIOR: Extra HP + defense. Once per combat you can POWER ATTACK for bonus damage.
ROGUE: Higher accuracy. At the beginning of most combats you get a free SNEAK ATTACK.
MAGE: 3 powerful spells with limited uses. Very strong when spells are available.
""")

    if topic in ("roll", "dice", "rolling"):
        print("""
HOW TO ROLL (this is what you asked about!)
Just type things like:
  roll d20
  2d6
  d8+3
  1d20 + 5

The game understands standard D&D dice notation.
You can do this at the main prompt or during combat to see the numbers.
""")

    print("Type 'help' again anytime. Type 'tutorial' for the new-player walkthrough.")
    print("=" * 56 + "\n")


def show_tutorial():
    print("""
QUICK NEW PLAYER / "IT'S BEEN YEARS" TUTORIAL

1. You just created a character with a class. Your class gives you special moves.

2. At the > prompt, the main things are:
   - Type "venture" (or just "v") to go deeper. This is how you progress.
   - Most of the time something will attack you.

3. When something attacks you, you enter COMBAT.
   - Type "attack" to swing your weapon.
   - Or type "cast magic missile" if you're a Mage.
   - Warriors and Rogues have their own special moves (power attack / sneak attack).

4. You can ALWAYS type "roll d20" or "roll 2d6" to roll dice yourself.
   This is great for learning or when you just want to see the numbers.

5. After fights you get loot. Type "inventory" to see it.
   Type "equip longsword" (or whatever you found) to use better gear.

6. Healing: Use potions from inventory or "rest".

7. Goal: Go deep (depth 8 is a win), kill things, get rich, don't die.

8. Stuck? Type "help" or "help combat".

The dice are brutal and fair. Have fun!
""")


# ====================== COMBAT ======================

def do_combat(player: Dict, monster: Dict) -> bool:
    print(f"\n>>> COMBAT: {player['name']} vs {monster['name']} <<<")
    print(f"You face {monster['desc']}.\n")
    speak(f"Combat! You face {monster['name']}.")

    m_hp = monster["hp"]
    m_ac = monster["ac"]
    m_dmg = monster["dmg"]

    p_hp = player["hp"]
    player["special_used"] = False

    # Class pre-combat bonus
    sneak_bonus = 0
    if player["class"] == "rogue" and random.random() < 0.85:
        sneak_bonus = roll(6)
        print(f"** SNEAK ATTACK! ** You slip in a surprise strike for {sneak_bonus} damage!")
        m_hp -= sneak_bonus
        if m_hp <= 0:
            print("The creature drops before it can react!")
            return _combat_victory(player, monster, m_hp)

    # Mage shield pre-buff possibility
    shield_active = False

    round_num = 1
    while p_hp > 0 and m_hp > 0:
        print(f"--- Round {round_num} ---")
        print(f"  You: {max(0, p_hp)}/{player['max_hp']} HP   |   {monster['name']}: {max(0, m_hp)} HP")

        # === Player decision ===
        action = ""
        while not action:
            cmd = input("Combat> ").strip()
            if handle_free_roll(cmd):
                continue
            action = parse_command(cmd)

            if action == "attack":
                break
            elif action == "cast" and "cast" in cmd.lower():
                spell_name = cmd.lower().replace("cast", "").strip()
                if player["class"] == "mage":
                    dmg, m_hp = _try_cast_spell(player, spell_name, m_hp)
                    if dmg is not None:
                        action = "cast"
                        break
                else:
                    print("Only mages can cast spells.")
            elif action == "power_attack" and player["class"] == "warrior":
                if not player["special_used"]:
                    player["special_used"] = True
                    print("** POWER ATTACK! ** You put everything into this swing.")
                    # Do a stronger hit
                    atk = roll(20, get_attack_bonus(player))
                    if atk >= m_ac:
                        base_dmg = roll(6, 2) + (get_attack_bonus(player) // 2)
                        m_hp -= base_dmg
                        print(f"  You smash for {base_dmg} damage!")
                    else:
                        print("  Even with extra effort you miss...")
                    action = "special"
                    break
                else:
                    print("You've already used your power attack this fight.")
            elif action == "sneak_attack" and player["class"] == "rogue":
                print("You already got your sneak attack in at the top of the fight (or try a normal attack).")
            elif action == "potion":
                if _try_drink_potion_in_combat(player):
                    pass
                else:
                    continue
            elif action == "flee":
                if random.randint(1, 20) >= 12:
                    print("You slip away into the darkness!")
                    player["hp"] = max(0, p_hp)
                    return False
                else:
                    print("You fail to escape!")
                    action = "flee_fail"
                    break
            elif action == "help":
                show_help("combat")
                continue
            elif action == "stats":
                _show_quick_stats(player)
                continue
            elif action == "roll":
                # already handled above
                continue
            else:
                print("In combat you can: attack, cast <spell>, power attack, potion, flee, or roll dice.")
                continue

        # Normal attack resolution
        if action == "attack":
            atk = roll(20, get_attack_bonus(player))
            if atk >= m_ac:
                dmg = roll(6, max(0, get_attack_bonus(player)))
                if atk >= 20:  # crit-ish
                    dmg = int(dmg * 1.8)
                    print(f"  CRITICAL! You roll {atk} and deal {dmg} damage!")
                else:
                    print(f"  You roll {atk} and hit for {dmg} damage!")
                m_hp -= dmg
            else:
                print(f"  You roll {atk}... and miss.")

        # Monster turn (unless dead)
        if m_hp > 0:
            mon_atk = roll(20)
            p_def = get_defense(player)
            if shield_active:
                p_def += 4
            if mon_atk >= p_def:
                dmg = m_dmg + (1 if monster["name"] == BOSS["name"] else 0)
                print(f"  The {monster['name']} hits you for {dmg} damage!")
                p_hp -= dmg
            else:
                print(f"  The {monster['name']} misses.")

        p_hp = max(0, p_hp)
        round_num += 1

        if p_hp <= 0:
            break

    player["hp"] = p_hp

    if p_hp <= 0:
        print(f"\nYou have been slain by the {monster['name']}...")
        return False

    return _combat_victory(player, monster, m_hp)


def _combat_victory(player: Dict, monster: Dict, final_m_hp: int) -> bool:
    xp = monster.get("xp", 10) + (player.get("depth", 0) * 2)
    gain_xp(player, xp)
    player["kills"] += 1
    print(f"\nYou defeated the {monster['name']}!")
    print(f"Gain {xp} experience.")

    # Loot
    gold = roll(6) + roll(4) + (player["depth"] // 2)
    player["gold"] += gold
    print(f"You scavenge {gold} gold.")

    # Chance for actual item
    if random.random() < 0.55:
        new_item = _make_item(random.choice([t["name"] for t in ITEM_TEMPLATES]))
        player["inventory"].append(new_item)
        print(f"You also find: {new_item['name']}! (type 'inventory' to see it)")

    # Mage spell refresh chance
    if player["class"] == "mage" and random.random() < 0.3:
        for s in player["spell_uses"]:
            player["spell_uses"][s] = min(SPELLS[s]["uses"], player["spell_uses"][s] + 1)
        print("The battle's magic lingers — you feel one of your spells refreshed slightly.")

    return True


def _try_cast_spell(player: Dict, spell_key: str, monster_hp: int) -> Tuple[Optional[int], int]:
    key = spell_key.lower().strip()
    if key not in player.get("spell_uses", {}):
        # try fuzzy match
        for real_key in SPELLS:
            if key in real_key or real_key in key:
                key = real_key
                break
        else:
            print(f"You don't know a spell called '{spell_key}'.")
            return None, monster_hp

    if player["spell_uses"].get(key, 0) <= 0:
        print(f"You have no uses left of {SPELLS[key]['name']}.")
        return None, monster_hp

    player["spell_uses"][key] -= 1
    spell = SPELLS[key]
    print(f"You cast {spell['name']}!")

    if spell["effect"] is None:  # Shield
        print("A shimmering force field springs up around you (+4 defense this fight).")
        # We handle this via a flag in the caller, but for simplicity we just note it
        # (The combat loop already supports shield_active conceptually; simplified here)
        return 0, monster_hp
    else:
        dmg, dmg_type = spell["effect"](player)
        print(f"The spell deals {dmg} {dmg_type} damage!")
        return dmg, monster_hp - dmg


def _try_drink_potion_in_combat(player: Dict) -> bool:
    for i, item in enumerate(player["inventory"]):
        if item["type"] == "consumable" and "heal" in item:
            heal = item["heal"] + roll(3)
            player["hp"] = min(player["max_hp"], player["hp"] + heal)
            print(f"You quaff a {item['name']} mid-battle and recover {heal} HP!")
            player["inventory"].pop(i)
            return True
    print("You have no healing potions.")
    return False


def _show_quick_stats(player: Dict):
    print(f"\nHP: {player['hp']}/{player['max_hp']} | Attack +{get_attack_bonus(player)} | Defense {get_defense(player)} | Gold {player['gold']}")


# ====================== MAIN GAME LOOP ======================

def game_loop(player: Dict):
    print("\nYou stand at the entrance of the ancient crypt.")
    print("Darkness, danger, and treasure await. Type 'help' if you need guidance.\n")

    while player["hp"] > 0:
        level_str = get_level_display(player)
        print(f"\n[ Depth {player['depth']}/{player['max_depth']} | {level_str} | HP: {player['hp']}/{player['max_hp']} | "
              f"Gold: {player['gold']} | Kills: {player['kills']} | {player['class_name']} ]")

        if player["depth"] >= player["max_depth"]:
            if player["difficulty"] == "insane" and player["depth"] >= 30:
                # Special final boss
                print("\nThe air grows deathly cold. The Drakulich rises from the deepest crypt!")
                print(r"""
      /\
     /  \
    | () |
     \  /
      \/
   [BLUE FLAME]
   (pixel eye)
""")
                drakulich = {
                    "name": "The Drakulich",
                    "hp": 80,
                    "ac": 19,
                    "dmg": 12,
                    "xp": 300,
                    "desc": "a colossal undead dragon-lord, wings like tattered sails, eyes burning with the malice of a thousand lost campaigns."
                }
                if not do_combat(player, drakulich):
                    return
                print("\nYou have slain the Drakulich! Legend will never forget this day.")
            else:
                print("\nYou have reached a depth few mortals ever see. The crypt itself seems to acknowledge you.")
            print("You decide this is far enough. Time to return with your legend.")
            break

        cmd = input("> ").strip()
        if not cmd:
            continue

        # Dice rolling always available
        if handle_free_roll(cmd):
            continue

        action = parse_command(cmd)

        if action == "venture":
            player["depth"] += 1
            depth_xp = 25 + (player["depth"] * 4)
            gain_xp(player, depth_xp)
            print(f"\nYou descend deeper... (Depth {player['depth']})  [+{depth_xp} XP]")
            time.sleep(0.35)
            flavor = random.choice(ROOM_FLAVORS)
            print(flavor)
            speak(flavor)

            if random.random() < 0.70:
                if player["difficulty"] == "insane" and player["depth"] >= 30:
                    mon = {
                        "name": "The Drakulich",
                        "hp": 85,
                        "ac": 20,
                        "dmg": 13,
                        "xp": 350,
                        "desc": "a colossal undead dragon-lord, wings like tattered sails, eyes burning with the malice of a thousand lost campaigns."
                    }
                    print("\nThe final horror awakens!")
                elif player["depth"] >= 7 and random.random() < 0.55:
                    mon = BOSS.copy()
                    print("\nA terrible presence fills the chamber...")
                else:
                    mon = random.choice(MONSTER_TABLE).copy()
                    # Scale for higher difficulties and depth
                    scale = player["depth"] // 3
                    if player["difficulty"] in ("hard", "insane"):
                        scale += 1 if player["difficulty"] == "insane" else 0
                    mon["hp"] += scale * 2
                    mon["ac"] += scale // 2
                    mon["dmg"] += scale // 2
                    mon["xp"] += scale * 3
                if not do_combat(player, mon):
                    return
            else:
                print("\nThe chamber is eerily quiet.")
                if random.random() < 0.4:
                    g = roll(8) + 3
                    player["gold"] += g
                    print(f"You find {g} gold hidden in the gloom.")

        elif action == "search":
            print("\nYou search the area thoroughly...")
            time.sleep(0.4)
            if random.random() < 0.6:
                gold = roll(10) + 3
                player["gold"] += gold
                print(random.choice(LOOT_DESCRIPTIONS) + f" (+{gold} gold)")
            else:
                print("Nothing... but your skin crawls.")
                if random.random() < 0.28:
                    dmg = roll(5)
                    player["hp"] -= dmg
                    print(f"A trap springs! You take {dmg} damage.")
                    if player["hp"] <= 0:
                        return

        elif action == "potion":
            if not any(i["type"] == "consumable" and "heal" in i for i in player["inventory"]):
                print("You have no healing potions.")
            else:
                use_consumable(player, "potion")

        elif action == "rest":
            heal = roll(5) + 1
            player["hp"] = min(player["max_hp"], player["hp"] + heal)
            print(f"\nYou bind your wounds and recover {heal} HP.")
            if random.random() < 0.38:
                print("Something heard you...")
                mon = random.choice(MONSTER_TABLE).copy()
                if not do_combat(player, mon):
                    return

        elif action == "inventory":
            show_inventory(player)

        elif action == "equip":
            # extract item name after the word equip/wield/wear
            parts = cmd.lower().split()
            item_name = " ".join(parts[1:]) if len(parts) > 1 else ""
            if item_name:
                equip_item(player, item_name)
            else:
                print("Equip what? (e.g. 'equip longsword')")
                show_inventory(player)

        elif action == "stats":
            print(f"\n=== {player['name']} the {player['class_name']} ===")
            print(f"Level: {player.get('level',1)}  (XP: {player['xp']}/{player['xp_to_next']})")
            print(f"HP: {player['hp']}/{player['max_hp']}")
            print(f"Attack bonus: +{get_attack_bonus(player)}")
            print(f"Defense: {get_defense(player)}")
            print(f"Gold: {player['gold']}   Kills: {player['kills']}")
            print(f"Depth: {player['depth']}/{player['max_depth']}  ({player['difficulty'].title()})")
            if player["class"] == "mage":
                print("Spell uses remaining:")
                for k, v in player["spell_uses"].items():
                    print(f"  {SPELLS[k]['name']}: {v}")

        elif action == "map":
            show_map(player)

        elif action == "flee":
            print("\nYou turn and head for the surface with what you've earned.")
            break

        elif action == "cast":
            if player["class"] == "mage":
                print("In combat you can cast. Out here, maybe later when something needs blasting.")
            else:
                print("Only mages cast spells.")

        elif action == "help":
            topic = ""
            if "combat" in cmd.lower():
                topic = "combat"
            elif "roll" in cmd.lower() or "dice" in cmd.lower():
                topic = "roll"
            elif "mage" in cmd.lower() or "spell" in cmd.lower():
                topic = "mage"
            show_help(topic)

        elif "speak on" in cmd or cmd == "tts on" or cmd == "voice on":
            toggle_tts(True)

        elif "speak off" in cmd or cmd == "tts off" or cmd == "voice off":
            toggle_tts(False)

        elif cmd.startswith("say "):
            text = cmd[4:].strip()
            if text:
                speak(text)
            else:
                print("Say what?")

        # === Save / Load commands ===
        elif cmd == "save" or cmd.startswith("save "):
            slot = cmd[5:].strip() if cmd.startswith("save ") else None
            save_game(player, slot)

        elif cmd == "load" or cmd.startswith("load "):
            slot = cmd[5:].strip() if cmd.startswith("load ") else None
            if not slot:
                saves = list_saves()
                if not saves:
                    print("No saved games found. Use 'save myname' to create one.")
                else:
                    print("Available saves:")
                    for i, s in enumerate(saves, 1):
                        print(f"  {i}. {s}")
                    choice = input("Enter number or exact name to load: ").strip()
                    if choice.isdigit():
                        idx = int(choice) - 1
                        if 0 <= idx < len(saves):
                            slot = saves[idx]
                    else:
                        slot = choice
            if slot:
                loaded = load_game(slot)
                if loaded:
                    # Replace current player with loaded one and continue
                    player.clear()
                    player.update(loaded)
                    print("Resuming your adventure...")
                    speak("Resuming adventure.")

        elif cmd in ("saves", "save list", "list saves"):
            saves = list_saves()
            if saves:
                print("Your saved games:")
                for s in saves:
                    print(f"  - {s}")
                print("Use 'load <name>' or 'load' to pick one.")
            else:
                print("No saves yet. Type 'save' to create one.")

        elif cmd in ("highscores", "scores", "leaderboard"):
            try:
                scores_file = os.path.join(SAVES_DIR, "highscores.json")
                if os.path.exists(scores_file):
                    with open(scores_file) as f:
                        scores = json.load(f)
                    print("TOP SCORES (local):")
                    for i, s in enumerate(scores[:4], 1):
                        print(f"  {i}. {s['name']} - {s['score']} pts (Lvl {s['level']}, {s['difficulty']} D{s['depth']})")
                else:
                    print("No high scores yet. Beat the dungeon first!")
            except:
                print("No high scores recorded.")

        elif cmd.startswith("delete save ") or cmd.startswith("del save "):
            slot = cmd.split("save ", 1)[1].strip()
            delete_save(slot)

        elif action == "quit":
            print("\nYou abandon the crypt.")
            break

        else:
            print("I don't understand that. Type 'help' for a list of commands.")
            # Easter egg: let them try weird things
            if random.random() < 0.1:
                print("(The shadows whisper: try 'roll d20' or 'venture' or 'map')")
            if "gygax" in cmd.lower() or "voice of the master" in cmd.lower():
                if random.random() < 0.6:
                    print("\nA strange die rolls out of the darkness... Gygax's Guiding Die!")
                    die = {"name": "Gygax's Guiding Die", "type": "trinket", "value": 60, "desc": "A twenty-sided die etched with tiny dungeon rooms instead of numbers. It whispers the wisdom of the Grandmaster."}
                    player["inventory"].append(die)
                    speak("The Grandmaster's die has found you.")
                else:
                    print("\nThe name echoes... but nothing answers this time.")

        if player["hp"] <= 0:
            break

    # End of run
    print("\n" + "=" * 56)
    print("                     ADVENTURE ENDS")
    print("=" * 56)
    print(f"  {player['name']} the {player['class_name']} (Level {player.get('level',1)})")
    print(f"  Deepest depth : {player['depth']}/{player.get('max_depth',8)} ({player.get('difficulty','easy')})")
    print(f"  Gold          : {player['gold']}")
    print(f"  Monsters slain: {player['kills']}")
    score = player['gold'] + (player['kills'] * 15) + (player['depth'] * 30) + (player.get('level',1)*25)
    print(f"  FINAL SCORE   : {score}")
    print("=" * 56)

    # Simple local scoreboard
    save_cli_high_score(score, player)

    if player["depth"] >= player.get('max_depth', 8):
        if player.get("difficulty") == "insane":
            print("\nA true legend of the crypt. The ice finally rests.")
            print("This was once Azara, a proud ice dragon. The lich curse would not release it.")
            print("You put an ancient wrong to rest. The cold breeze is gone.")
        else:
            print("\nA true legend of the crypt.")
    elif player["kills"] >= 8:
        print("\nThe bodies pile high in your wake.")
    else:
        print("\nYou survived. That counts for something.")

    speak(f"Adventure over. Your final score is {score}.")

def save_cli_high_score(score, player):
    try:
        ensure_saves_dir()
        scores_file = os.path.join(SAVES_DIR, "highscores.json")
        scores = []
        if os.path.exists(scores_file):
            with open(scores_file, "r") as f:
                scores = json.load(f)
        scores.append({
            "name": player.get("name", "Unknown"),
            "score": score,
            "level": player.get("level", 1),
            "difficulty": player.get("difficulty", "easy"),
            "depth": player.get("depth", 0),
            "date": time.strftime("%Y-%m-%d")
        })
        scores.sort(key=lambda x: x["score"], reverse=True)
        scores = scores[:10]
        with open(scores_file, "w") as f:
            json.dump(scores, f, indent=2)
        print("\nHigh scores updated. Use 'highscores' command next time.")
    except:
        pass


def main():
    try:
        while True:
            player = create_character()
            game_loop(player)

            again = input("\nTempt fate once more? (y/n) ").strip().lower()
            if again not in ("y", "yes", "yeah", "sure"):
                print("\nFarewell. May your future rolls be kinder.")
                break
            print("\n" + "="*60 + "\n")
    except KeyboardInterrupt:
        print("\n\nYou flee the crypt in panic. The darkness remains.")
        sys.exit(0)


if __name__ == "__main__":
    main()
