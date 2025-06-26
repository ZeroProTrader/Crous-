import requests
from bs4 import BeautifulSoup
import time
from datetime import datetime
import re

    # Ton token et le chat_id du groupe ou utilisateur
TELEGRAM_BOT_TOKEN = "token"
TELEGRAM_CHAT_ID = "id-chat"

    # URLs pour été 2025 et année universitaire 2025/2026
URLS = {
        "Été 2025": "https://trouverunlogement.lescrous.fr/tools/37/search",
        "Année 2025/2026": "https://trouverunlogement.lescrous.fr/tools/39/search"
    }
BASE_URL = "https://trouverunlogement.lescrous.fr"

def extract_ville(address):
        match = re.search(r"\b\d{5}\s+([A-Za-zÀ-ÿ\- ]+)", address)
        if match:
            ville_brute = match.group(1).strip()
            ville = re.sub(r"\bCedex\b.*", "", ville_brute, flags=re.IGNORECASE).strip()
            return ville.title()
        return "Ville inconnue"

def get_logements(source_name, search_url):
        page = 1
        logements = []

        while True:
            response = requests.get(f"{search_url}?page={page}")
            response.encoding = 'utf-8'
            soup = BeautifulSoup(response.text, "html.parser")
            cards = soup.select("li.fr-col-12.fr-col-sm-6")

            if not cards:
                break

            for card in cards:
                try:
                    title = card.select_one("h3.fr-card__title").text.strip()
                    lien = card.select_one("a")["href"]
                    adresse = card.select_one("p.fr-card__desc").text.strip()
                    logements.append((source_name, title, adresse, BASE_URL + lien))
                except Exception as e:
                    print(f"[WARN] Carte ignorée ({source_name}, page {page}) : {e}")
            page += 1

        return logements

def format_message(source, title, address, url):
        ville = extract_ville(address)
        return (f"{source}\n"
                f"*Ville:* {ville}\n"
                f"*Label:* {title}\n"
                f"*Address:* {address}\n"
                f"*Link:* {url}")

def send_telegram_message(message):
        if "PARIS" in message.upper():
            message = "🔴" + message + "🔴"

        url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
        payload = {
            "chat_id": TELEGRAM_CHAT_ID,
            "text": message,
            "parse_mode": "Markdown"
        }
        response = requests.post(url, json=payload)
        if response.status_code != 200:
            print("[ERROR] Envoi Telegram échoué :", response.text)

def send_startup_message():
        send_telegram_message(" - Ce qu’on propose :\n"
"-Alertes en temps réel sur les nouvelles offres de logements CROUS disponibles.\n"
"-Possibilité de personnaliser les alertes selon ta ville ou ta région préférée (Clermont-Ferrand, Lyon, Lille, Paris, etc).\n"
 "Service fiable, rapide.\n")

def monitor():
        seen = set()
        send_startup_message()
        first_run = True

        while True:
            now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            print(f"[{now}] Vérification des nouvelles offres...")

            try:
                all_logements = []
                for source_name, url in URLS.items():
                    logements = get_logements(source_name, url)
                    all_logements.extend(logements)

                if first_run:
                    seen.update(all_logements)
                    print(f"[{now}] [INIT] {len(all_logements)} logements enregistrés (toutes sources)")
                    first_run = False
                else:
                    new_logements = [log for log in all_logements if log not in seen]
                    if new_logements:
                        for source, title, adresse, url in new_logements:
                            message = format_message(source, title, adresse, url)
                            send_telegram_message(message)
                            seen.add((source, title, adresse, url))
                            print(f"[{now}] [NOUVEAU] Logement trouvé : {title} ({source})")
                            time.sleep(3)
                    else:
                        print(f"[{now}] [INFO] Aucun nouveau logement trouvé")

            except Exception as e:
                print(f"[{now}] [ERROR] Une erreur est survenue : {str(e)}")

            time.sleep(60)

if __name__ == "__main__":
        monitor()
