from flask import Flask, request, redirect, jsonify
import hashlib
import redis
import datetime

app = Flask(__name__)
redis_client = redis.StrictRedis(host='localhost', port=6379, db=0)

def generate_short_url(long_url):
    hash_object = hashlib.sha256(long_url.encode())
    short_url = hash_object.hexdigest()[:8]  # Using first 8 characters as the short URL
    return short_url

@app.route('/shorten', methods=['POST'])
def shorten_url():
    data = request.get_json()
    long_url = data.get('url')

    if not long_url:
        return jsonify({'error': 'URL is required'}), 400

    short_url = generate_short_url(long_url)
    redis_client.set(short_url, long_url)
    return jsonify({'short_url': short_url}), 201

@app.route('/<short_url>', methods=['GET'])
def redirect_to_long_url(short_url):
    long_url = redis_client.get(short_url)
    if not long_url:
        return jsonify({'error': 'Short URL not found'}), 404

    click_timestamp = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    redis_client.hincrby(short_url, 'clicks', 1)
    redis_client.zincrby('clicks_by_hour', 1, click_timestamp)

    return redirect(long_url.decode())

@app.route('/analytics/<short_url>', methods=['GET'])
def get_analytics(short_url):
    clicks = redis_client.hget(short_url, 'clicks')
    return jsonify({'clicks': int(clicks) if clicks else 0})

if __name__ == '__main__':
    app.run(debug=True)
