ChessGenius is a rudimentary JavaScript chess AI.

ChessGenius' core interest is the application's decision-making process. External libraries are used to implement all functions outside the scope of the AI:


Using the chessboard.js API to create a chessboard GUI
Using the chess.js API for games play

The AI employs the minimax method, which has been optimized by alpha-beta pruning.

The evaluation function adapts Sunfish.py's piece square tables and reduces the requirement for nested loops by updating the total based on each move rather than re-calculating the sum of individual pieces at each leaf node.

To display the 'advantage' bar, a global total is employed to maintain track of black's assessment score after each move.
