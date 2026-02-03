import sys, mysql.connector
from PyQt6 import QtWidgets, QtGui
from PyQt6.QtWidgets import *
from admin_ui import Ui_MainWindow as AdminUI
from auth_ui import Ui_MainWindow as AuthUI
from user_ui import Ui_MainWindow as UserUI


def db_query(query, params=None, fetch=False):
    conn = mysql.connector.connect(host="localhost", user="root", password="Tipelka.006", database="sample_db")
    cursor = conn.cursor()
    cursor.execute(query, params or ())
    if fetch:
        result = cursor.fetchall()
    else:
        conn.commit()
        result = None
    cursor.close()
    conn.close()
    return result


class AuthWindow(QtWidgets.QMainWindow):
    def __init__(self):
        super().__init__()
        self.ui = AuthUI()
        self.ui.setupUi(self)
        self.ui.pushButton.clicked.connect(self.login)
        self.admin_window = None
        self.user_window = None

    def login(self):
        login = self.ui.lineEdit.text()
        password = self.ui.lineEdit_2.text()

        if login == "admin" and password == "admin123":
            self.admin_window = AdminWindow()
            self.admin_window.show()
            self.close()
        elif login == "user" and password == "user123":
            self.user_window = UserWindow(login)
            self.user_window.show()
            self.close()
        else:
            QMessageBox.warning(self, "Ошибка", "Неверный логин или пароль")


class AdminWindow(QtWidgets.QMainWindow):
    def __init__(self):
        super().__init__()
        self.ui = AdminUI()
        self.ui.setupUi(self)
        self.ui.pushButton.clicked.connect(self.delete_record)
        self.ui.pushButton_2.clicked.connect(self.edit_record)
        self.ui.pushButton_3.clicked.connect(self.add_record)
        self.ui.pushButton_4.clicked.connect(self.filter_records)
        self.load_data()

    def load_data(self, filter_text=""):
        if filter_text:
            data = db_query("SELECT * FROM students WHERE first_name LIKE %s OR last_name LIKE %s OR age LIKE %s OR gpa LIKE %s",
                            (f"%{filter_text}%", f"%{filter_text}%", f"%{filter_text}%", f"%{filter_text}%"), fetch=True)
        else:
            data = db_query("SELECT * FROM students", fetch=True)
            
        model = QtGui.QStandardItemModel()
        model.setHorizontalHeaderLabels(['ID', 'Имя', 'Фамилия', 'Возраст', 'Средний балл'])
        for row in data:
            model.appendRow([QtGui.QStandardItem(str(col)) for col in row])
        self.ui.tableView.setModel(model)

    def delete_record(self):
        selected = self.ui.tableView.selectionModel().selectedRows()
        if selected:
            if QMessageBox.question(self, 'Подтверждение',
                                    'Удалить выбранную запись?') == QMessageBox.StandardButton.Yes:
                model = self.ui.tableView.model()
                for index in selected:
                    db_query("DELETE FROM students WHERE student_id = %s", (model.item(index.row(), 0).text(),))
                self.load_data()

    def add_record(self):
        dialog = RecordDialog(self)
        if dialog.exec():
            db_query("INSERT INTO students (first_name, last_name, age, gpa) VALUES (%s, %s, %s, %s)",
                     dialog.get_data())
            self.load_data()

    def edit_record(self):
        selected = self.ui.tableView.selectionModel().selectedRows()
        if selected:
            index = selected[0]
            model = self.ui.tableView.model()
            data = [model.item(index.row(), i).text() for i in range(1, 5)]
            dialog = RecordDialog(self, *data)
            if dialog.exec():
                db_query("UPDATE students SET first_name=%s, last_name=%s, age=%s, gpa=%s WHERE student_id=%s",
                         dialog.get_data() + (model.item(index.row(), 0).text(),))
                self.load_data()

    def filter_records(self):
        filter_text, ok = QInputDialog.getText(self, "Фильтр", "Введите текст для поиска:")
        if ok:
            self.load_data(filter_text)


class UserWindow(QtWidgets.QMainWindow):
    def __init__(self, login):
        super().__init__()
        self.ui = UserUI()
        self.ui.setupUi(self)
        # ИСПРАВЛЕНИЕ: Выбираем все столбцы, включая student_id
        data = db_query("SELECT student_id, first_name, last_name, age, gpa FROM students", fetch=True)
        # ИСПРАВЛЕНИЕ: Правильные заголовки
        model = QtGui.QStandardItemModel()
        model.setHorizontalHeaderLabels(['ID', 'Имя', 'Фамилия', 'Возраст', 'Средний балл'])
        for row in data:
            model.appendRow([QtGui.QStandardItem(str(col)) for col in row])
        self.ui.tableView.setModel(model)


class RecordDialog(QDialog):
    def __init__(self, parent=None, first_name="", last_name="", age="", gpa=""):
        super().__init__(parent)
        layout = QVBoxLayout()
        self.edits = [QLineEdit(text) for text in [first_name, last_name, age, gpa]]
        # ИСПРАВЛЕНИЕ: Правильные названия полей
        labels = ["Имя:", "Фамилия:", "Возраст:", "Средний балл:"]
        for label, edit in zip(labels, self.edits):
            layout.addWidget(QLabel(label))
            layout.addWidget(edit)
        button_layout = QHBoxLayout()
        for text, slot in [("OK", self.accept), ("Отмена", self.reject)]:
            btn = QPushButton(text)
            btn.clicked.connect(slot)
            button_layout.addWidget(btn)
        layout.addLayout(button_layout)
        self.setLayout(layout)

    def get_data(self):
        return tuple(edit.text() for edit in self.edits)


if __name__ == "__main__":
    app = QtWidgets.QApplication(sys.argv)
    auth_window = AuthWindow()
    auth_window.show()
    sys.exit(app.exec())



pip install pyqt6
pip install mysql-connector-python
pip install pymysql
pyuic6 admin.ui -o admin_ui.py
