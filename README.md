# ---------------- IMPORTS ---------        filename = ydl.prepare_filename(info)
        return filename, info.get("thumbnail"), info.get("title")

# ---------------- PLAYBACK ----------------
async def play_next(chat_id):
    if chat_id not in queues or queues[chat_id].empty():
        playing_status[chat_id] = False
        return

    url = await queues[chat_id].get()
    playing_status[chat_id] = True
    file_path, thumb_url, title = download_song(url)
    await app.send_photo(chat_id, photo=thumb_url, caption=f"🎶 Now Playing: {fancy_text(title)}", reply_markup=get_buttons())

    await pytgcalls.join_group_call(chat_id, InputStream(InputAudioStream(file_path)))
    os.remove(file_path)

# ---------------- HANDLERS ----------------
@app.on_message(filters.command("start"))
async def start(client, message):
    await message.reply_text(fancy_text("🎵 Fancy Music Bot is Online!"), reply_markup=get_buttons())

@app.on_message(filters.text & ~filters.bot)
async def detect_url(client, message):
    chat_id = message.chat.id
    url = message.text.strip()
    if chat_id not in queues:
        queues[chat_id] = asyncio.Queue()
    await queues[chat_id].put(url)
    if not playing_status.get(chat_id, False):
        await play_next(chat_id)
    else:
        await message.reply_text("Added to queue ✅")

@app.on_callback_query()
async def buttons(client, callback_query):
    chat_id = callback_query.message.chat.id
    action = callback_query.data
    if action == "pause": await pytgcalls.pause_stream(chat_id)
    elif action == "resume": await pytgcalls.resume_stream(chat_id)
    elif action == "skip":
        await pytgcalls.leave_group_call(chat_id)
        if not queues[chat_id].empty():
            await play_next(chat_id)
    elif action == "stop":
        await pytgcalls.leave_group_call(chat_id)
        queues[chat_id] = asyncio.Queue()
        playing_status[chat_id] = False
        
