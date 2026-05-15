# Whale-Rada
تحليل ومراقبه الاسهم وتعطيه تريدات يوميه
Whale Radar
# bot.py
import requests
import asyncio
import logging
from datetime import datetime
from telegram import Update
from telegram.ext import Application, CommandHandler, ContextTypes

# ---------- الإعدادات ----------
TELEGRAM_TOKEN = "8397228783:AAFlR_ccWx5848TFCotW5DZ1xGzMfHUxxn8"
TWELVE_DATA_API_KEY = "ae6cf1578f8c4f8faf5d81962372acd9"
WATCHLIST = ["COMI", "TMGH", "MFPC", "SWDY", "ASCM", "EKHO", "ABUK", "SKPC", "ESRS", "FWRY", "CCAP", "ORAS", "EAST", "EMFD", "OCDI"]

# إعدادات التسجيل (logs)
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# ---------- دوال جلب البيانات ----------
def get_stock_data(symbol):
    """جلب بيانات آخر يوم تداول من Twelve Data."""
    url = f"https://api.twelvedata.com/time_series?symbol={symbol}&exchange=EGX&interval=1day&outputsize=2&apikey={TWELVE_DATA_API_KEY}"
    try:
        resp = requests.get(url, timeout=10)
        data = resp.json()
        if data.get('status') == 'ok' and data.get('values'):
            last = data['values'][0]
            prev = data['values'][1] if len(data['values']) > 1 else last
            return {
                'symbol': symbol,
                'date': last['datetime'],
                'close': float(last['close']),
                'high': float(last['high']),
                'low': float(last['low']),
                'volume': float(last['volume']),
                'prev_close': float(prev['close'])
            }
    except Exception as e:
        logger.error(f"خطأ في جلب {symbol}: {e}")
    return None

def analyze_signal(data):
    """تحليل الشروط: طفرة حجم + إغلاق بالقرب من القمة + زخم صاعد."""
    if not data:
        return None
    high = data['high']
    low = data['low']
    close = data['close']
    volume = data['volume']
    prev_close = data['prev_close']
    # شروط الإشارة
    volume_spike = volume > 500000   # عتبة مبسطة (يمكن تحسينها لاحقاً)
    near_high = (close - low) > (high - low) * 0.7
    momentum_up = close > prev_close
    if volume_spike and near_high and momentum_up:
        entry = round(high * 1.001, 2)
        stop_loss = round(low * 0.99, 2)
        risk = round(entry - stop_loss, 2)
        tp1 = round(entry + risk, 2)
        tp2 = round(entry + risk * 2, 2)
        return {
            'entry': entry,
            'sl': stop_loss,
            'risk': risk,
            'tp1': tp1,
            'tp2': tp2
        }
    return None

def format_signal(symbol, data, indicators):
    """تنسيق رسالة الإشارة."""
    msg = f"🚀 *إشارة تداول جديدة لـ {symbol}*\n\n"
    msg += f"📅 التاريخ: {data['date']}\n"
    msg += f"💰 السعر الحالي: {data['close']}\n"
    msg += f"📈 أعلى سعر: {data['high']}  |  📉 أدنى سعر: {data['low']}\n"
    msg += f"📊 VWAP تقريبي: {round((data['high']+data['low']+data['close'])/3,2)}\n"
    msg += f"⚖️ حجم التداول: {int(data['volume']):,}\n\n"
    msg += f"🎯 *نقطة الدخول:* {indicators['entry']}\n"
    msg += f"🛑 *وقف الخسارة:* {indicators['sl']} (المخاطرة: {indicators['risk']} نقطة)\n"
    msg += f"💰 *الهدف الأول (1:1):* {indicators['tp1']}\n"
    msg += f"💰 *الهدف الثاني (1:2):* {indicators['tp2']}\n\n"
    msg += f"<i>انتظر كسر القمة {data['high']} في الجلسة الحالية لتأكيد الدخول.</i>"
    return msg

# ---------- أوامر البوت ----------
async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "مرحباً بك في بوت تحليل البورصة المصرية 📊\n\n"
        "الأوامر المتاحة:\n"
        "/scan - فحص فوري للأسهم وإرسال الإشارات\n"
        "/watchlist - عرض الأسهم المراقبة\n"
        "/help - عرض المساعدة"
    )

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "📌 *كيفية الاستخدام:*\n"
        "- استخدم /scan لبدء مسح جميع الأسهم وإرسال أي إشارات.\n"
        "- الإشارة تصدر عند توفر:\n"
        "   • حجم تداول مرتفع\n"
        "   • سعر الإغلاق قرب القمة\n"
        "   • زخم صاعد (أعلى من اليوم السابق)\n"
        "سيتم إرسال نقاط الدخول والوقف والأهداف.\n\n"
        "البوت يعمل بشكل تلقائي كل ساعة (يمكنك طلب فحص يدوي في أي وقت).",
        parse_mode='Markdown'
    )

async def watchlist_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    watchlist_str = "\n".join([f"- {s}" for s in WATCHLIST])
    await update.message.reply_text(f"📋 *قائمة الأسهم المراقبة:*\n{watchlist_str}", parse_mode='Markdown')

async def scan_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    await update.message.reply_text("🔍 جاري فحص الأسهم... قد يستغرق دقيقة.")
    signals_found = 0
    for symbol in WATCHLIST:
        data = get_stock_data(symbol)
        if data:
            indicators = analyze_signal(data)
            if indicators:
                msg = format_signal(symbol, data, indicators)
                await context.bot.send_message(chat_id=chat_id, text=msg, parse_mode='HTML')
                signals_found += 1
        await asyncio.sleep(0.5)  # تجنب تجاوز الحدود
    await update.message.reply_text(f"✅ اكتمل الفحص. تم العثور على {signals_found} إشارة.")

# ---------- الفحص الدوري التلقائي ----------
async def periodic_scan(context: ContextTypes.DEFAULT_TYPE):
    chat_id = context.job.chat_id
    signals_found = 0
    for symbol in WATCHLIST:
        data = get_stock_data(symbol)
        if data:
            indicators = analyze_signal(data)
            if indicators:
                msg = format_signal(symbol, data, indicators)
                await context.bot.send_message(chat_id=chat_id, text=msg, parse_mode='HTML')
                signals_found += 1
        await asyncio.sleep(0.5)
    logger.info(f"الفحص الدوري: {signals_found} إشارات جديدة.")

# ---------- تشغيل البوت ----------
def main():
    app = Application.builder().token(TELEGRAM_TOKEN).build()
    app.add_handler(CommandHandler("start", start_command))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("watchlist", watchlist_command))
    app.add_handler(CommandHandler("scan", scan_command))
    # جدولة فحص تلقائي كل ساعة (3600 ثانية)
    # تحتاج إلى معرف chat_id مسبقاً – يمكنك إرسال /start ثم أخذ chat_id من logs أو تخزينه.
    # للتبسيط، سنطلب من المستخدم تشغيل الفحص التلقائي بعد أول تفاعل.
    # بدلاً من ذلك يمكن استرجاع chat_id من قاعدة بيانات – لكن هنا سنعتمد على الأمر اليدوي فقط لتجنب التعقيد.
    # من الأفضل إضافة أمر /subscribe لتفعيل الفحص الدوري. سأضيفه.
    # سأضيف الأمر /subscribe لتفعيل الفحص الدوري:
    async def subscribe(update: Update, context: ContextTypes.DEFAULT_TYPE):
        chat_id = update.effective_chat.id
        # إزالة أي وظيفة قديمة
        current_jobs = context.job_queue.jobs()
        for job in current_jobs:
            if job.name == f"auto_scan_{chat_id}":
                job.schedule_removal()
        # إضافة وظيفة جديدة
        context.job_queue.run_repeating(periodic_scan, interval=3600, first=10, chat_id=chat_id, name=f"auto_scan_{chat_id}")
        await update.message.reply_text("✅ تم تفعيل الفحص التلقائي كل ساعة. ستصلك الإشارات تلقائياً.")
    app.add_handler(CommandHandler("subscribe", subscribe))
    app.run_polling()

if __name__ == "__main__":
    main()
