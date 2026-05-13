import sys
from datetime import datetime
from typing import List, Optional, Dict, Any


# Базовые классы исключений
class BankingException(Exception):
    """Базовое исключение банковской системы"""
    pass


class ValidationError(BankingException):
    """Ошибка валидации данных"""
    pass


class BusinessRuleError(BankingException):
    """Нарушение бизнес-правил"""
    pass


class InsufficientFundsError(BankingException):
    """Недостаточно средств"""
    pass


class CardBlockedError(BankingException):
    """Карта заблокирована"""
    pass


# Класс Owner (владелец карты)
class Owner:
    def __init__(self, last_name: str, first_name: str, phone: str):
        self.last_name = last_name
        self.first_name = first_name
        self.phone = phone
        self.pin = None
    
    def set_pin(self, pin: str):
        if not isinstance(pin, str) or len(pin) != 4 or not pin.isdigit():
            raise ValidationError("ПИН-код должен быть строкой из 4 символов")
        self.pin = pin
    
    def change_pin(self, old_pin: str, new_pin: str):
        if self.pin != old_pin:
            raise ValidationError("Введенный ПИН-код не соответствует текущему")
        self.set_pin(new_pin)


# Класс Account (счёт)
class Account:
    def __init__(self, owner: Owner, account_type: str = "debit"):
        self.owner = owner
        self.balance = 0.0
        self.account_type = account_type
        self.blocked = False
        self.interest_rate = 0.0
    
    def deposit(self, amount: float):
        if amount <= 0:
            raise ValidationError("Сумма должна быть положительной")
        self.balance += amount
    
    def withdraw(self, amount: float):
        if amount <= 0:
            raise ValidationError("Сумма должна быть положительной")
        if self.balance < amount:
            raise InsufficientFundsError("Недостаточно денег для осуществления перевода.")
        if self.account_type == "saving":
            raise BusinessRuleError("С накопительного счёта нельзя списывать покупки")
        self.balance -= amount


# Класс Card (карта)
class Card:
    MCC_RESTRICTED = {"4829", "7995"}  # Запрещённые MCC-коды
    
    def __init__(self, owner: Owner, card_type: str, payment_system: str, account: Account):
        self.owner = owner
        self.card_type = card_type
        self.payment_system = payment_system
        self.account = account
        self.blocked = False
        self.cashback_rate = 0.0
    
    def pay(self, amount: float, mcc: str):
        if self.blocked:
            raise CardBlockedError("Карта закрыта или заблокирована. Невозможно провести операцию.")
        
        if amount > 500000:
            self.blocked = True
            raise BusinessRuleError("Сумма оплаты превышает лимит. Операция заблокирована до выяснения причин.")
        
        if not mcc.isdigit() or len(mcc) != 4:
            raise ValidationError("Неверный код категории продавца (MCC)")
        
        if mcc in self.MCC_RESTRICTED:
            raise BusinessRuleError(f"Оплата отклонена. Покупки по {mcc} запрещены банком")
        
        self.account.withdraw(amount)
        
        # Начисление кешбэка
        if self.cashback_rate > 0:
            cashback = amount * self.cashback_rate
            self.account.balance += cashback
    
    def transfer(self, target_card: Optional['Card'], amount: float):
        if self.blocked:
            raise CardBlockedError("Карта закрыта или заблокирована. Невозможно провести операцию.")
        
        if target_card is None:
            raise ValidationError("Ошибка номером карты. Такой карты не существует.")
        
        if amount <= 0:
            raise ValidationError("Сумма должна быть положительной")
        
        if self.account.balance < amount:
            raise InsufficientFundsError("Недостаточно денег для осуществления перевода.")
        
        self.account.balance -= amount
        target_card.account.balance += amount


# Класс Bank
class Bank:
    def __init__(self, name: str, bik: str):
        self.name = name
        self.bik = bik
        self.clients: Dict[str, Owner] = {}  # телефон -> владелец
    
    def _check_phone_unique(self, last_name: str, first_name: str, phone: str):
        """Проверка уникальности телефона"""
        if phone in self.clients:
            existing_owner = self.clients[phone]
            if existing_owner.last_name != last_name or existing_owner.first_name != first_name:
                raise BusinessRuleError("Пользователь с таким телефоном уже зарегистрирован")
    
    def issue_simple_debit_card(self, last_name: str, first_name: str, pin: str, phone: str, payment_system: str) -> Card:
        self._check_phone_unique(last_name, first_name, phone)
        
        owner = Owner(last_name, first_name, phone)
        owner.set_pin(pin)
        
        account = Account(owner, "debit")
        card = Card(owner, "simple_debit", payment_system, account)
        
        if phone not in self.clients:
            self.clients[phone] = owner
        
        return card
    
    def issue_cashback_debit_card(self, last_name: str, first_name: str, pin: str, phone: str, payment_system: str) -> Card:
        self._check_phone_unique(last_name, first_name, phone)
        
        owner = Owner(last_name, first_name, phone)
        owner.set_pin(pin)
        
        account = Account(owner, "debit")
        card = Card(owner, "cashback_debit", payment_system, account)
        card.cashback_rate = 0.01  # 1% кешбэк
        
        if phone not in self.clients:
            self.clients[phone] = owner
        
        return card
    
    def issue_saving_card(self, last_name: str, first_name: str, pin: str, phone: str, payment_system: str, interest_rate: float) -> Card:
        self._check_phone_unique(last_name, first_name, phone)
        
        owner = Owner(last_name, first_name, phone)
        owner.set_pin(pin)
        
        account = Account(owner, "saving")
        account.interest_rate = interest_rate
        
        card = Card(owner, "saving", payment_system, account)
        
        if phone not in self.clients:
            self.clients[phone] = owner
        
        return card


# Декоратор для сбора ошибок
def collect_errors(func):
    """Декоратор, собирающий все ошибки вместо падения программы"""
    errors = []
    
    def wrapper(*args, **kwargs):
        nonlocal errors
        result = None
        try:
            result = func(*args, **kwargs)
        except BankingException as e:
            errors.append(str(e))
        return result
    
    wrapper.errors = errors
    return wrapper


# Функция для выполнения сценария и сбора ошибок
def execute_scenario(scenario_code: str) -> List[str]:
    """Выполняет код сценария и возвращает список ошибок"""
    errors = []
    local_vars = {
        'Bank': Bank,
        'Owner': Owner,
        'Account': Account,
        'Card': Card,
        'ValidationError': ValidationError,
        'BusinessRuleError': BusinessRuleError,
        'InsufficientFundsError': InsufficientFundsError,
        'CardBlockedError': CardBlockedError,
    }
    
    # Создаём специальный обработчик для перехвата исключений
    class ErrorCollector:
        def __init__(self):
            self.errors = []
        
        def __enter__(self):
            return self
        
        def __exit__(self, exc_type, exc_val, exc_tb):
            if exc_type is not None and issubclass(exc_type, BankingException):
                self.errors.append(str(exc_val))
                return True  # Подавляем исключение
            return False
    
    # Разбиваем код на отдельные выражения и выполняем их
    lines = scenario_code.strip().split('\n')
    
    for line in lines:
        line = line.strip()
        if not line or line.startswith('#'):
            continue
        
        # Выполняем каждую строку в контексте сборщика ошибок
        collector = ErrorCollector()
        with collector:
            try:
                exec(line, local_vars)
            except BankingException as e:
                errors.append(str(e))
        
        errors.extend(collector.errors)
    
    return errors


# Основная функция для обработки ввода
def main():
    # Читаем весь ввод
    input_lines = sys.stdin.read().strip().split('\n')
    
    # Первая строка - это команда для выполнения сценария
    scenario_code = '\n'.join(input_lines)
    
    # Выполняем сценарий
    errors = execute_scenario(scenario_code)
    
    # Выводим ошибки
    for error in errors:
        print(error)


if __name__ == "__main__":
    main()
