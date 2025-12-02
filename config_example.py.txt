import os

# REAL TOKEN & IDs (edit with your values)
BOT_TOKEN = os.getenv("BOT_TOKEN", "8584891759:AAGds400yEwDwk8LrqwiXLVyB5LxaTdMkrE")

# Main channel/group jahan creators & managers ka data forward hoga
MAIN_CHANNEL_ID = int(os.getenv("MAIN_CHANNEL_ID", "-1002863809955"))

# Admin private group jahan sheets, payments, week start/stop hoga
ADMIN_GROUP_ID = int(os.getenv("ADMIN_GROUP_ID", "-4992382090"))

# Admins list (comma separated IDs)
ADMIN_USER_IDS = [int(x) for x in os.getenv("ADMIN_USER_IDS", "5119859581").split(",")]

# Default minimum followers
MIN_FOLLOWERS_DEFAULT = int(os.getenv("MIN_FOLLOWERS_DEFAULT", "1000"))

