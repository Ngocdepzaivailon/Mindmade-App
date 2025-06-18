from flask import Flask, render_template, request, jsonify, session
from datetime import datetime
import database as db
import chatbot as cb
import json

app = Flask(__name__)
app.secret_key = 'your_secret_key_here'

# Trang chủ
@app.route('/')
def home():
    return render_template('index.html')

# Trắc nghiệm tâm lý
@app.route('/assessment')
def assessment():
    with open('static/data/questions.json') as f:
        questions = json.load(f)
    return render_template('assessment.html', questions=questions)

# Xử lý kết quả trắc nghiệm
@app.route('/submit_assessment', methods=['POST'])
def submit_assessment():
    answers = request.json
    score = calculate_score(answers)
    result = interpret_result(score)
    
    # Lưu kết quả vào database
    if 'user_id' in session:
        db.save_assessment(session['user_id'], score, result)
    
    return jsonify({'result': result})

# Nhật ký cảm xúc
@app.route('/diary')
def diary():
    if 'user_id' not in session:
        return redirect('/login')
    return render_template('diary.html')

# Lưu nhật ký
@app.route('/save_diary', methods=['POST'])
def save_diary():
    entry = {
        'date': datetime.now().strftime('%Y-%m-%d'),
        'mood': request.form.get('mood'),
        'content': request.form.get('content')
    }
    
    if 'user_id' in session:
        db.save_diary_entry(session['user_id'], entry)
    
    return jsonify({'status': 'success'})

# Chat với AI
@app.route('/chat')
def chat():
    return render_template('chat.html')

# Xử lý tin nhắn chat
@app.route('/send_message', methods=['POST'])
def send_message():
    message = request.form.get('message')
    response = cb.generate_response(message)
    return jsonify({'response': response})

# Tài nguyên học tập
@app.route('/resources')
def resources():
    with open('static/data/resources.json') as f:
        resources = json.load(f)
    return render_template('resources.html', resources=resources)

if __name__ == '__main__':
    app.run(debug=True)
    import random

class MindmateChatbot:
    def __init__(self):
        self.responses = {
            "stress": [
                "Mình hiểu bạn đang cảm thấy căng thẳng. Bạn có muốn chia sẻ thêm về điều gì đang khiến bạn stress không?",
                "Stress là phản ứng bình thường của cơ thể. Bạn thử hít thở sâu 5 lần xem sao nhé!"
            ],
            "depression": [
                "Nghe có vẻ bạn đang trải qua khoảng thời gian khó khăn. Bạn không đơn độc đâu.",
                "Những cảm xúc tiêu cực này rất khó chịu, nhưng chúng sẽ qua. Bạn đã ăn uống đủ chất chưa?"
            ],
            "anxiety": [
                "Lo lắng quá mức có thể khiến chúng ta mệt mỏi. Bạn thử tập trung vào hơi thở xem sao?",
                "Đôi khi viết ra những điều khiến bạn lo lắng sẽ giúp giảm bớt cảm giác này."
            ]
        }
        
        self.general_responses = [
            "Mình đang lắng nghe bạn. Bạn có thể nói thêm được không?",
            "Cảm ơn bạn đã chia sẻ. Điều này nghe có vẻ rất quan trọng với bạn.",
            "Mình hiểu những gì bạn đang trải qua. Bạn cảm thấy thế nào về điều đó?",
            "Đôi khi nói ra những suy nghĩ của mình có thể giúp chúng ta cảm thấy nhẹ nhõm hơn."
        ]
    
    def detect_keywords(self, message):
        message = message.lower()
        if 'stress' in message or 'căng thẳng' in message:
            return 'stress'
        elif 'depress' in message or 'trầm cảm' in message or 'buồn' in message:
            return 'depression'
        elif 'lo lắng' in message or 'lo âu' in message or 'hồi hộp' in message:
            return 'anxiety'
        return None
    
    def generate_response(self, message):
        keyword = self.detect_keywords(message)
        
        if keyword and random.random() > 0.3:  # 70% trả lời theo chủ đề
            return random.choice(self.responses[keyword])
        else:
            return random.choice(self.general_responses)

# Khởi tạo chatbot
chatbot = MindmateChatbot()

def generate_response(message):
    return chatbot.generate_response(message)
    import sqlite3
from datetime import datetime

def init_db():
    conn = sqlite3.connect('mindmate.db')
    c = conn.cursor()
    
    # Tạo bảng người dùng
    c.execute('''CREATE TABLE IF NOT EXISTS users
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                 username TEXT UNIQUE,
                 password TEXT,
                 age INTEGER,
                 created_at TIMESTAMP)''')
    
    # Tạo bảng trắc nghiệm
    c.execute('''CREATE TABLE IF NOT EXISTS assessments
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                 user_id INTEGER,
                 score INTEGER,
                 result TEXT,
                 date TIMESTAMP,
                 FOREIGN KEY(user_id) REFERENCES users(id))''')
    
    # Tạo bảng nhật ký
    c.execute('''CREATE TABLE IF NOT EXISTS diaries
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                 user_id INTEGER,
                 date DATE,
                 mood TEXT,
                 content TEXT,
                 FOREIGN KEY(user_id) REFERENCES users(id))''')
    
    conn.commit()
    conn.close()

def save_assessment(user_id, score, result):
    conn = sqlite3.connect('mindmate.db')
    c = conn.cursor()
    c.execute("INSERT INTO assessments (user_id, score, result, date) VALUES (?, ?, ?, ?)",
              (user_id, score, result, datetime.now()))
    conn.commit()
    conn.close()

def save_diary_entry(user_id, entry):
    conn = sqlite3.connect('mindmate.db')
    c = conn.cursor()
    c.execute("INSERT INTO diaries (user_id, date, mood, content) VALUES (?, ?, ?, ?)",
              (user_id, entry['date'], entry['mood'], entry['content']))
    conn.commit()
    conn.close()

def get_user_diary_entries(user_id):
    conn = sqlite3.connect('mindmate.db')
    c = conn.cursor()
    c.execute("SELECT date, mood, content FROM diaries WHERE user_id=? ORDER BY date DESC", (user_id,))
    entries = c.fetchall()
    conn.close()
    return entries

# Khởi tạo database khi import
init_db()
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mindmate - Người bạn tâm lý</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
    <header>
        <div class="container">
            <h1>Mindmate</h1>
            <nav>
                <a href="/">Trang chủ</a>
                <a href="/assessment">Trắc nghiệm</a>
                <a href="/diary">Nhật ký</a>
                <a href="/chat">Trò chuyện</a>
                <a href="/resources">Tài nguyên</a>
            </nav>
        </div>
    </header>
    
    <main class="container">
        {% block content %}{% endblock %}
    </main>
    
    <footer>
        <div class="container">
            <p>Mindmate - Người bạn đồng hành tâm lý của bạn</p>
            <p>Nếu bạn đang gặp khủng hoảng nghiêm trọng, vui lòng liên hệ đường dây nóng: 111</p>
        </div>
    </footer>
    
    <script src="{{ url_for('static', filename='js/main.js') }}"></script>
</body>
</html>
{% extends "base.html" %}

{% block content %}
<div class="chat-container">
    <h2>Trò chuyện với Mindmate</h2>
    <div class="chat-box" id="chatBox">
        <!-- Tin nhắn sẽ hiển thị ở đây -->
        <div class="bot-message">Xin chào! Mình là Mindmate, người bạn tâm lý của bạn. Hôm nay bạn thế nào?</div>
    </div>
    
    <div class="input-area">
        <input type="text" id="userInput" placeholder="Viết tin nhắn của bạn ở đây...">
        <button id="sendButton">Gửi</button>
    </div>
</div>

<script>
document.getElementById('sendButton').addEventListener('click', sendMessage);
document.getElementById('userInput').addEventListener('keypress', function(e) {
    if (e.key === 'Enter') sendMessage();
});

function sendMessage() {
    const userInput = document.getElementById('userInput');
    const message = userInput.value.trim();
    
    if (message) {
        // Hiển thị tin nhắn người dùng
        const chatBox = document.getElementById('chatBox');
        const userMsg = document.createElement('div');
        userMsg.className = 'user-message';
        userMsg.textContent = message;
        chatBox.appendChild(userMsg);
        
        // Xóa input
        userInput.value = '';
        
        // Cuộn xuống dưới cùng
        chatBox.scrollTop = chatBox.scrollHeight;
        
        // Gửi tin nhắn đến server và nhận phản hồi
        fetch('/send_message', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded',
            },
            body: 'message=' + encodeURIComponent(message)
        })
        .then(response => response.json())
        .then(data => {
            // Hiển thị phản hồi từ bot
            const botMsg = document.createElement('div');
            botMsg.className = 'bot-message';
            botMsg.textContent = data.response;
            chatBox.appendChild(botMsg);
            chatBox.scrollTop = chatBox.scrollHeight;
        });
    }
}
</script>
{% endblock %}
