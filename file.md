from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime
import utils

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///database.db'
db = SQLAlchemy(app)

class Task(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    description = db.Column(db.String(256), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    due_time = db.Column(db.DateTime, nullable=True)
    is_completed = db.Column(db.Boolean, default=False)

    def to_dict(self):
        return {
            'description': self.description,
            'created_at': self.created_at.strftime('%Y-%m-%d %H:%M'),
            'due_time': self.due_time.strftime('%Y-%m-%d %H:%M') if self.due_time else None,
            'is_completed': self.is_completed
        }

with app.app_context():
    db.create_all()

def handle_add_task(description):
    task = Task(description=description)
    db.session.add(task)
    db.session.commit()
    return f"Задача '{description}' добавлена."

def handle_view_tasks():
    tasks = Task.query.all()
    if not tasks:
        return "У вас нет задач."
    response = "Ваши задачи:\n"
    for idx, task in enumerate(tasks, 1):
        status = "✅ Выполнена" if task.is_completed else "❌ Не выполнена"
        due = f" до {task.due_time.strftime('%H:%M')}" if task.due_time else ""
        response += f"{idx}. {task.description}{due} - {status}\n"
    return response

def handle_complete_task(task_number):
    tasks = Task.query.all()
    if 0 < task_number <= len(tasks):
        task = tasks[task_number - 1]
        task.is_completed = True
        db.session.commit()
        return f"Задача '{task.description}' отмечена как выполненная."
    else:
        return "Неверный номер задачи."

@app.route('/', methods=['POST'])
def main():
    req = request.json

    # Обработка запроса от Алисы
    if req['session']['new']:
        welcome_text = "Привет! Я ваш помощник для управления задачами. Чем могу помочь?"
        return jsonify({
            "version": req['version'],
            "session": req['session'],
            "response": {
                "text": welcome_text,
                "end_session": False
            }
        })

    intent = req['request']['command'].lower()

    if "добавь задачу" in intent:
        description = intent.replace("добавь задачу", "").strip()
        response_text = handle_add_task(description)
    elif "покажи задачи" in intent:
        response_text = handle_view_tasks()
    elif "отметь задачу" in intent:
        try:
            task_number = int(intent.replace("отметь задачу", "").strip())
            response_text = handle_complete_task(task_number)
        except ValueError:
            response_text = "Пожалуйста, укажите номер задачи."
    else:
        response_text = "Извините, я не поняла вашу команду."

    return jsonify({
        "version": req['version'],
        "session": req['session'],
        "response": {
            "text": response_text,
            "end_session": False
        }
    })

if __name__ == '__main__':
    app.run()
