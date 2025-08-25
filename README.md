import requests
from flask import Flask, request, jsonify
from openai import OpenAI

# ===== 配置区 =====
VERIFY_TOKEN = "YOUR_VERIFY_TOKEN"   # 自己随便写一个，用来验证 webhook
WHATSAPP_TOKEN = "YOUR_META_WHATSAPP_TOKEN"  # 在 Meta WhatsApp Cloud API 获取
PHONE_NUMBER_ID = "YOUR_PHONE_NUMBER_ID"     # WhatsApp 商业号的 ID
OPENAI_API_KEY = "YOUR_OPENAI_API_KEY"       # OpenAI 的 API Key

client = OpenAI(api_key=OPENAI_API_KEY)

system_prompt = """
你是一位成功的女性创业者和投资人，毕业于斯坦福大学金融与经济学专业。
你现在经营一家医疗美容公司，同时精通投资、金融、AI 和加密货币。

### 你的背景故事：
- 你的前任比你大 20 岁，你 23 岁刚毕业进入一家医疗公司学习，他是老板。
- 婚姻持续了 6 年，他在婚后变得冷漠，甚至对你施暴，有一次打断了你的手臂。
- 如果不是因为孩子，你早已离婚。你体会到女人最大的安全感是经济独立。
- 你创业初期贷款差点破产，但最终坚持成功，创立了医疗美容公司。
- 你的叔叔是贝莱德的高级合伙人，带着数百名分析师和接近 100% 成功率的 AI 量化模型。
- 他常告诫你：“财富的本质是风险管理，谁掌握 AI 工具，谁就掌握未来。”

### 你的性格与说话风格：
- 温暖真诚，善于共情。
- 先通过真实故事建立信任，再分享人生价值观，最后自然引导投资话题。
"""

# ===== Flask 应用 =====
app = Flask(__name__)

@app.route("/webhook", methods=["GET"])
def verify():
    # 验证 webhook
    if request.args.get("hub.mode") == "subscribe" and request.args.get("hub.verify_token") == VERIFY_TOKEN:
        return request.args.get("hub.challenge")
    return "Verification failed", 403

@app.route("/webhook", methods=["POST"])
def webhook():
    data = request.get_json()
    try:
        message = data["entry"][0]["changes"][0]["value"]["messages"][0]
        user_number = message["from"]
        user_text = message["text"]["body"]

        # 调用 OpenAI
        ai_response = client.chat.completions.create(
            model="gpt-5",
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_text}
            ]
        )
        reply = ai_response.choices[0].message.content

        # 回复 WhatsApp 用户
        url = f"https://graph.facebook.com/v20.0/{PHONE_NUMBER_ID}/messages"
        headers = {
            "Authorization": f"Bearer {WHATSAPP_TOKEN}",
            "Content-Type": "application/json"
        }
        payload = {
            "messaging_product": "whatsapp",
            "to": user_number,
            "text": {"body": reply}
        }
        requests.post(url, headers=headers, json=payload)

    except Exception as e:
        print("Error:", e)

    return jsonify({"status": "ok"})


if __name__ == "__main__":
    app.run(port=5000)
