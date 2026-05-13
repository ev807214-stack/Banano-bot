mkdir banano-bot
cd banano-bot


npm init -y


npm install @whiskeysockets/baileys pino qrcode-terminal mongoose dotenv


nano index.js


nano .env


MONGO_URL=TU_MONGO_URL_AQUI
OWNER=TU_NUMERO@s.whatsapp.net


require("dotenv").config()

const {
  default: makeWASocket,
  useMultiFileAuthState,
  DisconnectReason
} = require("@whiskeysockets/baileys")

const mongoose = require("mongoose")
const P = require("pino")
const qrcode = require("qrcode-terminal")

mongoose.connect(process.env.MONGO_URL)

// ===== DATABASE =====
const userSchema = new mongoose.Schema({
  id: String,
  coins: { type: Number, default: 0 },
  bank: { type: Number, default: 0 },
  waifus: { type: Array, default: [] }
})

const User = mongoose.model("User", userSchema)

async function getUser(id) {
  let user = await User.findOne({ id })
  if (!user) user = await User.create({ id })
  return user
}

function rand(min, max) {
  return Math.floor(Math.random() * (max - min)) + min
}

// ===== WAIFUS =====
const waifus = [
  { name: "Zero Two", rarity: "legendaria", img: "https://i.imgur.com/0e5V8XH.jpg" },
  { name: "Rem", rarity: "epica", img: "https://i.imgur.com/8w1Q5pQ.jpg" },
  { name: "Asuna", rarity: "rara", img: "https://i.imgur.com/3g7kX9a.jpg" }
]

function randomWaifu() {
  return waifus[Math.floor(Math.random() * waifus.length)]
}

// ===== BOT =====
async function start() {

  const { state, saveCreds } = await useMultiFileAuthState("session")

  const sock = makeWASocket({
    logger: P({ level: "silent" }),
    auth: state
  })

  sock.ev.on("creds.update", saveCreds)

  sock.ev.on("connection.update", ({ qr, connection, lastDisconnect }) => {
    if (qr) qrcode.generate(qr, { small: true })

    if (connection === "close") {
      const reconnect =
        lastDisconnect?.error?.output?.statusCode !== DisconnectReason.loggedOut
      if (reconnect) start()
    }
  })

  sock.ev.on("messages.upsert", async ({ messages }) => {

    const msg = messages[0]
    if (!msg.message || msg.key.fromMe) return

    const from = msg.key.remoteJid
    const text =
      msg.message.conversation ||
      msg.message.extendedTextMessage?.text ||
      ""

    const user = await getUser(from)

    const owner = process.env.OWNER
    const isOwner = msg.key.participant === owner

    // ===== MENU =====
    if (text === "/menu") {
      await sock.sendMessage(from, {
        text: `🍌 BANANO PRO BOT

💰 Coins: ${user.coins}
🏦 Banco: ${user.bank}
🎴 Waifus: ${user.waifus.length}

💰 /trabajar
🏦 /depositar
🏦 /retirar
💸 /transferir
🦹 /robar
🎰 /ruleta
🎴 /waifu
🏆 /top
🖼️ /pin
👢 /kick`
      })
    }

    // ===== ECONOMÍA =====
    if (text === "/trabajar") {
      user.coins += rand(50, 200)
      await user.save()
    }

    if (text.startsWith("/depositar")) {
      const a = parseInt(text.split(" ")[1])
      if (user.coins >= a) {
        user.coins -= a
        user.bank += a
        await user.save()
      }
    }

    if (text.startsWith("/retirar")) {
      const a = parseInt(text.split(" ")[1])
      if (user.bank >= a) {
        user.bank -= a
        user.coins += a
        await user.save()
      }
    }

    // ===== WAIFU =====
    if (text === "/waifu") {
      const w = randomWaifu()
      user.waifus.push(w.name)
      await user.save()

      await sock.sendMessage(from, {
        image: { url: w.img },
        caption: `🎴 ${w.name}\n⭐ ${w.rarity}`
      })
    }

    // ===== PIN =====
    if (text.startsWith("/pin")) {
      const q = text.split(" ").slice(1).join(" ") || "anime"
      const url = `https://source.unsplash.com/800x600/?${encodeURIComponent(q)}`

      await sock.sendMessage(from, {
        image: { url },
        caption: `🖼️ ${q}`
      })
    }

    // ===== TOP =====
    if (text === "/top") {
      const all = await User.find().sort({ coins: -1 }).limit(10)

      let txt = "🏆 TOP BANANOCOINS\n\n"

      all.forEach((u, i) => {
        txt += `${i + 1}. ${u.id.split("@")[0]} - 💰 ${u.coins}\n`
      })

      await sock.sendMessage(from, { text: txt })
    }

    // ===== KICK =====
    if (text.startsWith("/kick")) {
      if (!isOwner) {
        return sock.sendMessage(from, { text: "❌ Solo owner" })
      }

      const target =
        msg.message.extendedTextMessage?.contextInfo?.participant

      if (!target) return

      await sock.groupParticipantsUpdate(from, [target], "remove")
    }
  })
}

start()


node index.js
