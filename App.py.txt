from flask import Flask, render_template, request
import random

app = Flask(__name__)

def parse_data(raw_data):
    lines = raw_data.strip().split('\n')
    data = []
    for i in range(0, len(lines), 3):
        timestamp = int(lines[i])
        number = int(lines[i+1])
        size = lines[i+2]
        data.append((timestamp, number, size))
    return data

def build_transition_matrix(data):
    transitions = {'Big': {'Big':0, 'Small':0}, 'Small': {'Big':0, 'Small':0}}
    counts = {'Big': 0, 'Small': 0}
    for i in range(len(data)-1):
        curr = data[i][2]
        nxt = data[i+1][2]
        transitions[curr][nxt] += 1
        counts[curr] += 1
    for curr in transitions:
        total = counts[curr]
        if total > 0:
            for nxt in transitions[curr]:
                transitions[curr][nxt] /= total
        else:
            transitions[curr]['Big'] = 0.5
            transitions[curr]['Small'] = 0.5
    return transitions

def get_number_lists(data):
    big_nums = set()
    small_nums = set()
    for _, number, size in data:
        if size == 'Big':
            big_nums.add(number)
        else:
            small_nums.add(number)
    return sorted(big_nums), sorted(small_nums)

def next_size(current_size, transitions):
    r = random.random()
    cumulative = 0
    for size, prob in transitions[current_size].items():
        cumulative += prob
        if r <= cumulative:
            return size
    return 'Big'

def next_number(size, big_nums, small_nums):
    if size == 'Big' and big_nums:
        return random.choice(big_nums)
    elif size == 'Small' and small_nums:
        return random.choice(small_nums)
    else:
        return random.randint(1, 9)

def predict_next(data, predict_count=10):
    transitions = build_transition_matrix(data)
    big_nums, small_nums = get_number_lists(data)
    last_timestamp, _, last_size = data[-1]
    predictions = []
    current_size = last_size
    current_timestamp = last_timestamp + 1
    for _ in range(predict_count):
        current_size = next_size(current_size, transitions)
        number = next_number(current_size, big_nums, small_nums)
        predictions.append((current_timestamp, number, current_size))
        current_timestamp += 1
    return predictions

@app.route('/', methods=['GET', 'POST'])
def index():
    predictions = []
    if request.method == 'POST':
        try:
            raw_data = request.form['data']
            count = int(request.form['count'])
            parsed = parse_data(raw_data)
            predictions = predict_next(parsed, count)
        except Exception as e:
            predictions = [("Error:", str(e), "")]
    return render_template('index.html', predictions=predictions)

if __name__ == '__main__':
    app.run()