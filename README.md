from random import randint
class Dot:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

    def __repr__(self):
        return f"({self.x}, {self.y})"

class BoardException(Exception):
#общий класс, содержит в себе все остальные виды исключений
    pass

class BoardOutException(BoardException):
    def __str__(self):
        return "Вы пытаетесь выстрелить за доску!"

class BoardUsedException(BoardException):
    def __str__(self):
        return "Вы уже стреляли в эту клетку"

class BoardWrongShipException(BoardException):
#исключение только для того, что бы размещать нормально корабли
    pass

class Ship: #класс корабля
    def __init__(self, bow, l, o):
        #конструктор в котором описанны поля которые нам нужны далее
        self.bow = bow
        self.l = l
        self.o = o
        self.lives = l

    @property #метод который вычисляет свойство
    def dots(self):
        ship_dots = []#список в котором точки всего корабля
        for i in range(self.l):#цикл от 0 до длинны корабля - 1
            cur_x = self.bow.x
            cur_y = self.bow.y

            if self.o == 0:
                cur_x += i

            elif self.o == 1:
                cur_y += i

            ship_dots.append(Dot(cur_x, cur_y))

        return ship_dots

    def shooten(self, shot):
    #показывает попали ли мы в корабль
        return shot in self.dots


class Board:#класс игровое поле
    def __init__(self, hid=False, size=6):
        self.size = size #размер поля
        self.hid = hid #видимость поля

        self.count = 0 #количество пораженных кораблей

        self.field = [["O"] * size for _ in range(size)]#состояние клетки

        self.busy = []# занятые точки (кораблем или вывстрелом)
        self.ships = []# список кораблей доски

    def add_ship(self, ship): # проверяет что каждая точка корабля не выходт за границы и не занята, иначе исключение!

        for d in ship.dots:
            if self.out(d) or d in self.busy:
                raise BoardWrongShipException()
        for d in ship.dots: # проходим по всем точкам корабля и ставим квадрат, так же добавляем в список занятых
            self.field[d.x][d.y] = "■"
            self.busy.append(d)

        self.ships.append(ship) # добавляем список собственных кораблей
        self.contour(ship) # обводим его по контуру

    def contour(self, ship, verb=False): # метот помечает соседние клетки от кораблей, для того что бы нельзя было поставить корабли рядом
        near = [
            (-1, -1), (-1, 0), (-1, 1),
            (0, -1), (0, 0), (0, 1),
            (1, -1), (1, 0), (1, 1)
        ]
        for d in ship.dots:
            for dx, dy in near:
                cur = Dot(d.x + dx, d.y + dy)
                if not (self.out(cur)) and cur not in self.busy:
                    if verb:
                        self.field[cur.x][cur.y] = "."
                    self.busy.append(cur)

    def __str__(self):
        res = "" # переменная в которую записывается вся доска
        res += "  | 1 | 2 | 3 | 4 | 5 | 6 |"
        for i, row in enumerate(self.field):# вывод доски построчно
            res += f"\n{i + 1} | " + " | ".join(row) + " |"

        if self.hid:# если hid - true, то символы кораблей заменяются на пустые
            res = res.replace("■", "O")
        return res

    def out(self, d):# проверяет находится ли точка за пределами доски
        return not ((0 <= d.x < self.size) and (0 <= d.y < self.size))

    def shot(self, d): # выстрел
        if self.out(d): # если точка выходит за границы
            raise BoardOutException()

        if d in self.busy: # если точка уже занята
            raise BoardUsedException()

        self.busy.append(d) # говорим что точка занята, теперь

        for ship in self.ships:
            if d in ship.dots: # если корабль подстрелен
                ship.lives -= 1 # уменьшаем длинну корабля
                self.field[d.x][d.y] = "X" # ставим Х на место ранения
                if ship.lives == 0:
                    self.count += 1
                    self.contour(ship, verb=True)
                    print("Корабль уничтожен!")
                    return False # возвращаем False т.к. дальше нет хода
                else:
                    print("Корабль ранен!")
                    return True

        self.field[d.x][d.y] = "."
        print("Мимо!")
        return False

    def begin(self): # когда начинаем игру, список бизи надо обнулить
        self.busy = []

    def defeat(self):
        return self.count == len(self.ships)


class Player:
    def __init__(self, board, enemy):
        self.board = board
        self.enemy = enemy

    def ask(self): # при попытке вызвать его будет еррор, он должен быть у потомков этого класса
        raise NotImplementedError()

    def move(self): # пытаемся сделать выстрел в бесконечном цикле
        while True:
            try:
                target = self.ask()
                repeat = self.enemy.shot(target)
                return repeat
            except BoardException as e:
                print(e)


class AI(Player):
    def ask(self):
        d = Dot(randint(0, 5), randint(0, 5))
        print(f"Ход компьютера: {d.x + 1} {d.y + 1}")
        return d


class User(Player):
    def ask(self):
        while True:
            cords = input("Ваш ход: ").split()

            if len(cords) != 2:
                print(" Введите 2 координаты! ")
                continue

            x, y = cords

            if not (x.isdigit()) or not (y.isdigit()):
                print(" Введите числа! ")
                continue

            x, y = int(x), int(y)

            return Dot(x - 1, y - 1)


class Game:
    def __init__(self, size=6):
        self.size = size
        pl = self.random_board() # генерируем доску для игорока
        co = self.random_board() # для компьютера
        co.hid = True # корабли компьютера скрыты

        self.ai = AI(co, pl) # игрок принявший доску co
        self.us = User(pl, co) # игрок принявший доску pl

    def random_board(self):
        board = None
        while board is None:
            board = self.random_place()
        return board

    def random_place(self): # ставим корабли
        lens = [3, 2, 2, 1, 1, 1, 1]
        board = Board(size=self.size)
        attempts = 0
        for l in lens:# в бесконечном цикле для каждой длинны корабля
            while True:
                attempts += 1
                if attempts > 2000: # если количетво попыток создать доску больше 2 тысяч вернем НОН
                    return None
                ship = Ship(Dot(randint(0, self.size), randint(0, self.size)), l, randint(0, 1))
                try:
                    board.add_ship(ship)
                    break
                except BoardWrongShipException:
                    pass
        board.begin()
        return board

    def greet(self):
        print("-------------------")
        print("  Приветсвуем вас  ")
        print("      в игре       ")
        print("    морской бой    ")
        print("-------------------")
        print(" формат ввода: x y ")
        print(" x - номер строки  ")
        print(" y - номер столбца ")

    def loop(self): # игровой цикл
        num = 0
        while True: # если номер хода четный - пользователь, иначе комп
            print("-" * 20)
            print("Доска пользователя:")
            print(self.us.board)
            print("-" * 20)
            print("Доска компьютера:")
            print(self.ai.board)
            if num % 2 == 0:
                print("-" * 20)
                print("Ходит пользователь!")
                repeat = self.us.move() # нужно ли повторить ход
            else:
                print("-" * 20)
                print("Ходит компьютер!")
                repeat = self.ai.move()
            if repeat:
                num -= 1

            if self.ai.board.defeat(): # количетсво пораженных короблей
                print("-" * 20)
                print("Пользователь выиграл!")
                break

            if self.us.board.defeat():
                print("-" * 20)
                print("Компьютер выиграл!")
                break
            num += 1

    def start(self):
        self.greet()
        self.loop()


g = Game()
g.start()
