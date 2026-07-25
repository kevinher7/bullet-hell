# Bullet Hell

The current repo is part of my adventure on learning low level programing from c++ by making a game. This first game is an ikaruga-like bullet hell game.

## Development Philosophy

This is a learning excercise, and as such, I will be the one doing the coding. You are simply my partner who mainly does
- Teaching of concepts, libraries or functionality
- Help brainstorming and grilling decisions I make 

I do the coding, you help me take decisions and understand the code. In this way I can make sure I clearly understand what I am doing while having you as a true assitant

## Some decisions

- I am going to develop a testing suite that takes json as game inputs in order to facilitate testing and reproducibility.
  - This also enables agents to play the game and test theories edge cases
- Since we want to render a lot of bullets we want to profile performance issues with tools like Instruments

## Main goals

- Having a playable game that feels fluid to play
- Render 10,000+ bullets at 60 FPS

