import random
import heapq

class Room:
    def __init__(self):
        self.wumpus = False
        self.pit = False
        self.gold = False
        self.visited = False

class Agent:
    def __init__(self):
        self.position = (0, 3)  # 시작 위치를 왼쪽 하단으로 설정
        self.direction = 'E'  # E: East, W: West, N: North, S: South
        self.arrows = 2
        self.has_gold = False
        self.path = []
        self.visited = set()
        self.visited.add(self.position)
        self.wumpus_positions = set()
        self.pit_positions = set()

    def turn_left(self):
        directions = ['N', 'W', 'S', 'E']
        self.direction = directions[(directions.index(self.direction) + 1) % 4]

    def turn_right(self):
        directions = ['N', 'W', 'S', 'E']
        self.direction = directions[(directions.index(self.direction) - 1) % 4]

    def go_forward(self, grid_size):
        x, y = self.position
        if self.direction == 'N':
            new_position = (x, y + 1)
        elif self.direction == 'S':
            new_position = (x, y - 1)
        elif self.direction == 'E':
            new_position = (x + 1, y)
        elif self.direction == 'W':
            new_position = (x - 1, y)

        if 0 <= new_position[0] < grid_size and 0 <= new_position[1] < grid_size:
            self.position = new_position
            self.visited.add(new_position)
            return False  # 벽에 부딪히지 않음
        else:
            return True  # 벽에 부딪힘
        
    def remove_gold(self, position):
        x, y = position
        self.grid[y][x].gold = False
        
    def grab(self):
        self.has_gold = True
        
    def climb(self):
        return self.position == (0, 3) and self.has_gold  # 시작 위치로 돌아와서 금을 가졌을 때 탈출

    def follow_path(self):
        if self.path:
            self.position = self.path.pop(0)

    def avoid_danger(self, percepts):
        if percepts['Breeze']:
            self.pit_positions.update(self.adjacent_positions(*self.position))
        if percepts['Stench']:
            self.wumpus_positions.update(self.adjacent_positions(*self.position))

    def adjacent_positions(self, x, y):
        positions = []
        if x > 0:
            positions.append((x - 1, y))
        if x < 3:  # assuming a 4x4 grid
            positions.append((x + 1, y))
        if y > 0:
            positions.append((x, y - 1))
        if y < 3:  # assuming a 4x4 grid
            positions.append((x, y + 1))
        return positions

class Environment:
    def __init__(self, grid_size=4):
        self.grid_size = grid_size
        self.grid = [[Room() for _ in range(grid_size)] for _ in range(grid_size)]
        self.agent = Agent()
        self.place_items()

    def random_position(self):
        while True:
            pos = (random.randint(0, self.grid_size - 1), random.randint(0, self.grid_size - 1))
            if pos != (0, 3):  # 시작 위치가 아니어야 함
                return pos

    def place_items(self):
        # Place Wumpus
        wumpus_pos = self.random_position()
        self.grid[wumpus_pos[1]][wumpus_pos[0]].wumpus = True

        # Place pits
        for _ in range(int(self.grid_size * self.grid_size * 0.1)):
            pit_pos = self.random_position()
            while self.grid[pit_pos[1]][pit_pos[0]].wumpus or (pit_pos == (0, 3)):  # 시작 위치에는 구덩이를 두지 않음
                pit_pos = self.random_position()
            self.grid[pit_pos[1]][pit_pos[0]].pit = True

        # Place gold
        gold_pos = self.random_position()
        while self.grid[gold_pos[1]][gold_pos[0]].wumpus or self.grid[gold_pos[1]][gold_pos[0]].pit or (gold_pos == (0, 3)):
            gold_pos = self.random_position()
        self.grid[gold_pos[1]][gold_pos[0]].gold = True

    def display_grid(self):
        for y in range(self.grid_size):
            for x in range(self.grid_size):
                if self.agent.position == (x, y):
                    print('A', end=' ')
                elif self.grid[y][x].wumpus:
                    print('W', end=' ')
                elif self.grid[y][x].pit:
                    print('P', end=' ')
                elif self.grid[y][x].gold:
                    print('G', end=' ')
                else:
                    print('.', end=' ')
            print()

    def get_percepts(self):
        x, y = self.agent.position
        room = self.grid[y][x]

        percepts = {
            'Stench': any(self.grid[ny][nx].wumpus for nx, ny in self.adjacent_positions(x, y)),
            'Breeze': any(self.grid[ny][nx].pit for nx, ny in self.adjacent_positions(x, y)),
            'Glitter': room.gold,
            'Wumpus': room.wumpus,
            'Pit': room.pit
        }
        return percepts

    def adjacent_positions(self, x, y):
        positions = []
        if x > 0:
            positions.append((x - 1, y))
        if x < self.grid_size - 1:
            positions.append((x + 1, y))
        if y > 0:
            positions.append((x, y - 1))
        if y < self.grid_size - 1:
            positions.append((x, y + 1))
        return positions

    def heuristic(self, a, b):
        return abs(a[0] - b[0]) + abs(a[1] - b[1])

    def a_star_search(self, start, goal, avoid_positions):
        frontier = []
        heapq.heappush(frontier, (0, start))
        came_from = {start: None}
        cost_so_far = {start: 0}

        while frontier:
            _, current = heapq.heappop(frontier)

            if current == goal:
                break

            for next in self.adjacent_positions(*current):
                if next in avoid_positions:
                    continue
                new_cost = cost_so_far[current] + 1
                if next not in cost_so_far or new_cost < cost_so_far[next]:
                    cost_so_far[next] = new_cost
                    priority = new_cost + self.heuristic(goal, next)
                    heapq.heappush(frontier, (priority, next))
                    came_from[next] = current

        path = []
        current = goal
        while current != start:
            path.append(current)
            current = came_from[current]
        path.reverse()
        return path

    def step(self):
        # Get percepts
        percepts = self.get_percepts()
        print(f"현재 위치: {self.agent.position}, 방향: {self.agent.direction}, 감각 정보: {percepts}")

        if percepts['Wumpus'] or percepts['Pit']:
            # 괴물이나 구덩이를 만났을 때
            print("괴물이나 구덩이를 만나 에이전트가 사망합니다.")
            return True  # Exploration ends
        elif percepts['Glitter']:
            # 금을 발견했을 때
            self.agent.grab()  # 금을 잡음
            print("금 발견! 금을 집었습니다.")
            # 시작 위치로 돌아가는 경로 탐색
            self.agent.position = (0, 3)
            return True
        elif self.agent.path:
            # 경로가 있을 때
            self.agent.follow_path()
            # 경로를 따르는 도중에 구덩이나 괴물을 만나는지 확인
            percepts = self.get_percepts()
            if percepts['Wumpus'] or percepts['Pit']:
                print("경로를 따르던 중 괴물이나 구덩이를 만나 에이전트가 사망합니다.")
                return True
            print(f"경로를 따라 이동: {self.agent.position}")
        else:
            # 감각 정보를 이용해 위험 피하기
            self.agent.avoid_danger(percepts)
            avoid_positions = self.agent.wumpus_positions.union(self.agent.pit_positions)

            # 그 외의 경우에는 앞으로 이동
            if not self.agent.go_forward(self.grid_size):
                print(f"앞으로 이동: {self.agent.position}")
            else:
                print("앞으로 이동 불가, 방향 변경")
                self.agent.turn_left()

        return False  # Exploration continues

def main():
    environment = Environment()  # 환경 생성
    steps = 0

    # 에이전트가 금을 찾아 탈출하거나 최대 100단계까지 시뮬레이션
    while not environment.agent.climb() and steps < 100:
        environment.display_grid()
        if environment.step():
            break
        steps += 1



if __name__ == "__main__":
    main()


