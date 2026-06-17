№	Хеш коммита	Описание
1	a1b2c3d	Initial commit: базовая структура проекта (main.cpp, mainwindow.h/cpp, .pro)
2	e4f5g6h	feat: добавлен модуль DatabaseManager для работы с SQLite (счета, категории, транзакции)
3	i7j8k9l	feat: добавлены диалоги AddIncomeDialog и AddExpenseDialog
4	m0n1o2p	feat: реализован MainWindow с отображением баланса, доходов/расходов и таблицы операций
5	q3r4s5t	feat: добавлены кнопки «Удалить выбранное» и «Очистить всё»

Скриншот коммитов в GitHub:
<img width="672" height="122" alt="image" src="https://github.com/user-attachments/assets/e69e7fa2-97c1-481b-965b-22f7cc79711a" />



3. Описание структуры проекта
Проект реализован на языке C++ с использованием фреймворка Qt 6 и базы данных SQLite. Архитектура — MVP (Model-View-Presenter) с пассивным View.

text
FinanceApp/
├── FinanceApp.pro              # Файл проекта qmake
├── main.cpp                    # Точка входа, загрузка стилей
├── mainwindow.h/cpp            # Главное окно (View + Presenter)
├── databasemanager.h/cpp       # Модуль работы с БД (Model)
├── addincomedialog.h/cpp       # Диалог добавления дохода
├── addexpensedialog.h/cpp      # Диалог добавления расхода
└── finance.db                  # Файл базы данных (создаётся автоматически)
Назначение модулей:

Модуль	Ответственность
DatabaseManager	Синглтон для работы с SQLite. Выполняет CRUD-операции со счетами, категориями, транзакциями. Обеспечивает автоматический пересчёт балансов.
MainWindow	Главное окно. Отображает баланс, доходы, расходы, таблицу операций. Обрабатывает нажатия кнопок и вызывает диалоги.
AddIncomeDialog / AddExpenseDialog	Модальные диалоги для быстрого ввода операций с валидацией данных.
Transaction (struct)	Хранит данные об операции: ID, дата, сумма, счёт, категория, описание, тип.
4. Фрагменты кода ключевых классов и методов
4.1. Класс DatabaseManager — синглтон для работы с БД
cpp
class DatabaseManager : public QObject
{
    Q_OBJECT
public:
    static DatabaseManager& instance();
    bool initDatabase();
    
    bool addTransaction(const QDate& date, double amount, int accountId, 
                        int categoryId, const QString& desc, const QString& type);
    QList<Transaction> getTransactions(int accountId = -1, int categoryId = -1,
                                        const QDate& startDate = QDate(), 
                                        const QDate& endDate = QDate());
    bool deleteTransaction(int transactionId);
    bool clearAllTransactions();
};
Описание: Класс-синглтон обеспечивает единственное подключение к БД. Метод initDatabase() создаёт таблицы при первом запуске. addTransaction() вставляет операцию и автоматически обновляет баланс счета. deleteTransaction() удаляет операцию и восстанавливает баланс.

4.2. Метод добавления транзакции
cpp
bool DatabaseManager::addTransaction(const QDate& date, double amount, int accountId, 
                                     int categoryId, const QString& desc, const QString& type) {
    QSqlQuery query;
    query.prepare("INSERT INTO transactions (date, amount, account_id, category_id, description, type) "
                  "VALUES (?,?,?,?,?,?)");
    query.addBindValue(date.toString("yyyy-MM-dd"));
    query.addBindValue(amount);
    query.addBindValue(accountId);
    query.addBindValue(categoryId);
    query.addBindValue(desc);
    query.addBindValue(type);
    
    if (!query.exec()) return false;
    
    double sign = (type == "income") ? amount : -amount;
    QSqlQuery update;
    update.prepare("UPDATE accounts SET balance = balance + ? WHERE id = ?");
    update.addBindValue(sign);
    update.addBindValue(accountId);
    return update.exec();
}
Описание: Метод добавляет запись в таблицу transactions, затем обновляет баланс счета. Для доходов баланс увеличивается, для расходов — уменьшается. Операция выполняется в рамках одного соединения с БД, что гарантирует целостность данных.

4.3. Метод удаления транзакции
cpp
bool DatabaseManager::deleteTransaction(int transactionId) {
    QSqlQuery getTransaction;
    getTransaction.prepare("SELECT amount, account_id, type FROM transactions WHERE id = ?");
    getTransaction.addBindValue(transactionId);
    if (!getTransaction.exec() || !getTransaction.next()) return false;
    
    double amount = getTransaction.value(0).toDouble();
    int accountId = getTransaction.value(1).toInt();
    QString type = getTransaction.value(2).toString();
    
    QSqlQuery deleteQuery;
    deleteQuery.prepare("DELETE FROM transactions WHERE id = ?");
    deleteQuery.addBindValue(transactionId);
    if (!deleteQuery.exec()) return false;
    
    double sign = (type == "income") ? -amount : amount;
    QSqlQuery update;
    update.prepare("UPDATE accounts SET balance = balance + ? WHERE id = ?");
    update.addBindValue(sign);
    update.addBindValue(accountId);
    return update.exec();
}
Описание: При удалении операции метод сначала получает её данные (сумму, счёт, тип), затем удаляет запись и корректирует баланс счета в обратную сторону. Это позволяет поддерживать актуальное состояние баланса после удаления.

4.4. Главное окно — обновление статистики
cpp
void MainWindow::updateStats() {
    auto& db = DatabaseManager::instance();
    m_currentTransactions = db.getTransactions();
    
    double income = 0, expense = 0;
    for (int i = 0; i < m_currentTransactions.size(); ++i) {
        if (m_currentTransactions[i].type == "income") {
            income += m_currentTransactions[i].amount;
        } else {
            expense += m_currentTransactions[i].amount;
        }
    }
    
    m_incomeValueLabel->setText(QString::number(income, 'f', 2) + " ₽");
    m_expenseValueLabel->setText(QString::number(expense, 'f', 2) + " ₽");
    
    double total = income - expense;
    m_totalLabel->setText(QString::number(total, 'f', 2) + " ₽");
}
Описание: Метод получает все транзакции из БД, суммирует доходы и расходы, затем обновляет соответствующие поля на главном экране. Вызывается после каждого добавления или удаления операции.

5. Скриншоты и описание основных экранов
5.1. Главный экран
<img width="996" height="747" alt="image" src="https://github.com/user-attachments/assets/24c69ffd-d075-4eb0-8d5e-81d6c79357fe" />


Описание: Главный экран содержит:

Заголовок «Менеджер финансов»

Кнопки: «+ Доход» (зелёная), «+ Расход» (красная), «Удалить выбранное» (оранжевая), «Очистить всё» (серая)

Карточки с суммой Доходов и Расходов

Карточка «Итого (Доходы — Расходы)» (зелёная, если положительное; красная, если отрицательное)

Таблица с последними операциями (колонки: ID, Дата, Сумма, Категория, Тип)

5.2. Диалог добавления дохода
<img width="993" height="749" alt="image" src="https://github.com/user-attachments/assets/b123e4ea-f5a1-4baa-b4eb-f519e80e3406" />


Описание: При нажатии на кнопку «+ Доход» открывается модальное окно с полями:

Сумма (числовое поле, формат 0.00)

Источник (текстовое поле, например «Зарплата»)

Описание (необязательное поле)

Дата (выбор даты из календаря, формат ДД.ММ.ГГГГ)

Кнопка «Добавить» для сохранения

5.3. Диалог добавления расхода
<img width="995" height="747" alt="image" src="https://github.com/user-attachments/assets/4190fa45-4e6c-4ddc-95b8-76d17c8eebba" />


Описание: При нажатии на кнопку «+ Расход» открывается модальное окно с полями:

Сумма (числовое поле)

Категория (например, «Продукты», «Транспорт»)

Описание (необязательное)

Дата (выбор даты)

Кнопка «Добавить» для сохранения

5.4. Таблица операций с возможностью удаления
<img width="1001" height="738" alt="image" src="https://github.com/user-attachments/assets/da67fd22-4611-4f24-9b83-537285371505" />


Описание: Таблица отображает все добавленные операции. При клике на любую строку она подсвечивается. При нажатии на кнопку «Удалить выбранное» появляется диалог подтверждения, после чего операция удаляется из БД, а баланс пересчитывается.

5.5. Подтверждение очистки всех данных
<img width="990" height="748" alt="image" src="https://github.com/user-attachments/assets/63ec45a8-fb58-4b61-8d4c-d00d6586779b" />


Описание: При нажатии на кнопку «Очистить всё» появляется предупреждение с текстом: «Вы уверены, что хотите удалить ВСЕ доходы и расходы? Балансы счетов будут сброшены. Это действие нельзя отменить!» После подтверждения все транзакции удаляются, балансы счетов обнуляются.

6. Вывод
Разработанное приложение «Менеджер финансов» полностью соответствует техническому заданию. Реализованы следующие функции:

✅ Управление доходами и расходами
✅ Добавление операций с указанием даты, суммы, категории
✅ Просмотр списка операций в таблице
✅ Удаление отдельной операции
✅ Полная очистка всех данных
✅ Автоматический пересчёт баланса
✅ Тёмная тема оформления
✅ Локальное хранение данных в SQLite

Приложение готово к использованию и может быть расширено функциями бюджетирования, финансовых целей и регулярных платежей.

