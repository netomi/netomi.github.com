---
layout: project
title: Tic Tac Toe
description: simple Tic-Tac-Toe android app with unbeatable AI
img: /img/logo-tictactoe.png
link: https://github.com/netomi/tic-tac-toe
role: owner
license: Apache license 2.0
category: game
tags: android mvvm minimax_ai
---

The simple board game _Tic-Tac-Toe_ served as an exercise for my first Android application.

The application has been developed using the MVVM (Model-View-View Model) pattern. The actual model class
provides access to the state of the current Tic-Tac-Toe game via events published via rxjava.

The view model observes the game model and controls the actual view displaying the state of the game. Inputs from
human players or the computer AI are fed into the model to advance the game.

As a result, the whole game logic is fully reactive, either based on user input or events generated from the game model.

Additionally, the human player can compete against an unbeatable AI using the minimax algorithm with alpha-beta pruning.