from flask import Flask, render_template, request, redirect, url_for
import instaloader
import os

app = Flask(__name__)

# Initialize Instaloader
L = instaloader.Instaloader()

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    password = request.form['password']
    profile_name = request.form['profile']

    # Attempt login with provided credentials
    try:
        L.login(username, password)
    except instaloader.exceptions.TwoFactorAuthRequiredException:
        return redirect(url_for('two_factor', username=username, password=password, profile=profile_name))

    return redirect(url_for('scrape', profile=profile_name))

@app.route('/two_factor')
def two_factor():
    username = request.args.get('username')
    password = request.args.get('password')
    profile = request.args.get('profile')
    return render_template('two_factor.html', username=username, password=password, profile=profile)

@app.route('/two_factor', methods=['POST'])
def two_factor_post():
    username = request.form['username']
    password = request.form['password']
    profile_name = request.form['profile']
    two_factor_code = request.form['two_factor_code']

    L.two_factor_login(two_factor_code)

    return redirect(url_for('scrape', profile=profile_name))

@app.route('/scrape')
def scrape():
    profile_name = request.args.get('profile')
    profile = instaloader.Profile.from_username(L.context, profile_name)

    # Create a directory to save the posts
    if not os.path.exists(profile_name):
        os.makedirs(profile_name)

    # Scrape the first 10 posts from the profile
    post_count = 10
    for post in profile.get_posts():
        if post_count == 0:
            break
        L.download_post(post, target=profile_name)
        post_count -= 1

    return f'Downloaded {10 - post_count} posts from the profile {profile_name}'

if __name__ == '__main__':
    app.run(debug=True)
