# Fruit-Falling-game.fla
Fruit Falling Game is a fast-paced arcade game made in Adobe Animate (AS3). Catch falling fruits, avoid bombs, and score high! Move the basket left or right to collect fruits. Each fruit boosts your score, while bombs and misses cost points. Speed increases for an exciting challenge!
Open FRUIT_FALLING_GAME.fla in Adobe Animate (AS3).

// Main.as
package {
    import flash.display.MovieClip;
    import flash.events.Event;
    import flash.events.TimerEvent;
    import flash.events.KeyboardEvent;
    import flash.ui.Keyboard;
    import flash.utils.Timer;
    import flash.text.TextField;

    public class Main extends MovieClip {
        // stage instances (must exist on stage)
        public var basket_mc:MovieClip;      // instance name on stage
        public var score_txt:TextField;      // dynamic textfield on stage
        public var gameover_txt:TextField;   // optional, show on game over

        private var spawnTimer:Timer;
        private var items:Array;             // Array of FallingItem (FruitItem/BombItem)
        private var score:int = 0;
        private var dropInterval:int = 900;  // ms between spawns (decreases over time)
        private var speedMultiplier:Number = 1.0;
        private var elapsedSeconds:int = 0;
        private var difficultyTimer:Timer;

        private var moveSpeed:Number = 10;   // speed when using keyboard
        private var isLeftDown:Boolean = false;
        private var isRightDown:Boolean = false;

        public function Main() {
            // ensure stage is available
            if (stage) init();
            else addEventListener(Event.ADDED_TO_STAGE, init);
        }

        private function init(e:Event = null):void {
            if (e) removeEventListener(Event.ADDED_TO_STAGE, init);

            // init variables
            items = [];
            score = 0;
            updateScoreText();

            // if gameover_txt provided, hide initially
            if (gameover_txt) {
                gameover_txt.visible = false;
            }

            // spawn timer
            spawnTimer = new Timer(dropInterval);
            spawnTimer.addEventListener(TimerEvent.TIMER, onSpawnTimer);
            spawnTimer.start();

            // difficulty timer: every 5 seconds increase difficulty slightly
            difficultyTimer = new Timer(5000);
            difficultyTimer.addEventListener(TimerEvent.TIMER, onDifficultyTick);
            difficultyTimer.start();

            // game loop
            addEventListener(Event.ENTER_FRAME, onEnterFrame);

            // keyboard controls
            stage.addEventListener(KeyboardEvent.KEY_DOWN, onKeyDown);
            stage.addEventListener(KeyboardEvent.KEY_UP, onKeyUp);

            // mouse movement (move basket to mouse x)
            stage.addEventListener(Event.MOUSE_LEAVE, onMouseLeave);
            stage.addEventListener(Event.MOUSE_MOVE, onMouseMove);
        }

        private function onMouseMove(e:Event):void {
            // Direct following — smoother for touch/mouse devices
            if (stage) {
                // lock basket to inside stage bounds
                var targetX:Number = stage.mouseX;
                basket_mc.x = clamp(targetX, basket_mc.width/2, stage.stageWidth - basket_mc.width/2);
            }
        }
        private function onMouseLeave(e:Event):void {
            // nothing, but prevents errors if mouse leaves stage
        }

        private function onKeyDown(e:KeyboardEvent):void {
            if (e.keyCode == Keyboard.LEFT || e.keyCode == Keyboard.A) isLeftDown = true;
            if (e.keyCode == Keyboard.RIGHT || e.keyCode == Keyboard.D) isRightDown = true;
        }
        private function onKeyUp(e:KeyboardEvent):void {
            if (e.keyCode == Keyboard.LEFT || e.keyCode == Keyboard.A) isLeftDown = false;
            if (e.keyCode == Keyboard.RIGHT || e.keyCode == Keyboard.D) isRightDown = false;
        }

        private function onEnterFrame(e:Event):void {
            // update basket with keyboard input (mouse still works if moved)
            if (isLeftDown) basket_mc.x = Math.max(basket_mc.width/2, basket_mc.x - moveSpeed);
            if (isRightDown) basket_mc.x = Math.min(stage.stageWidth - basket_mc.width/2, basket_mc.x + moveSpeed);

            // update each falling item
            for (var i:int = items.length - 1; i >= 0; i--) {
                var it:* = items[i];
                it.update(); // calls internal movement

                // check collision with basket
                if (it.hitTestObject(basket_mc)) {
                    // typed: check for isBomb flag
                    if (it.isBomb) {
                        // bomb -> reduce score and show effect
                        addScore(-10);
                        // optional: show explosion animation by playing a frame label or dispatching event
                    } else {
                        // fruit -> reward
                        addScore(5);
                    }
                    // remove item
                    removeItemAt(i);
                    continue;
                }

                // check if item hit ground (off bottom of stage)
                if (it.y > stage.stageHeight + 50) {
                    // missed fruit costs points; missed bomb is ignored (or small penalty)
                    if (!it.isBomb) {
                        addScore(-2);
                    }
                    removeItemAt(i);
                }
            }
        }

        private function onSpawnTimer(e:TimerEvent):void {
            spawnItem();
        }

        private function spawnItem():void {
            if (!stage) return;

            // random: 0..1 -> choose bomb chance (start low, increase later)
            var bombChance:Number = Math.min(0.15, 0.05 + speedMultiplier * 0.02); // up to ~15%
            var r:Number = Math.random();
            var item:FallingItem;
            if (r < bombChance) {
                item = new BombItem();
            } else {
                item = new FruitItem();
            }

            // position at random x along top, respect stage bounds
            var minX:Number = item.width/2;
            var maxX:Number = stage.stageWidth - item.width/2;
            item.x = minX + Math.random() * (maxX - minX);
            item.y = -item.height - (Math.random() * 50); // start slightly above stage

            // set speed based on difficulty multiplier
            item.fallSpeed = 3 * speedMultiplier + Math.random() * 2;
            item.rotationSpeed = (Math.random() - 0.5) * 6;

            addChild(item);
            items.push(item);
        }

        // increase difficulty periodically
        private function onDifficultyTick(e:TimerEvent):void {
            elapsedSeconds += 5;
            // every 10 seconds we make drops faster and spawn faster
            speedMultiplier += 0.08; // gently increase falling speed
            // reduce spawn interval down to a limit
            if (dropInterval > 300) {
                dropInterval = Math.max(300, dropInterval - 50);
                spawnTimer.delay = dropInterval;
            }
        }

        private function addScore(delta:int):void {
            score += delta;
            if (score < 0) score = 0;
            updateScoreText();
            // optional: show score pop up or sound
        }

        private function updateScoreText():void {
            if (score_txt) score_txt.text = "Score: " + score.toString();
        }

        private function removeItemAt(index:int):void {
            var it:FallingItem = items[index];
            if (it && contains(it)) removeChild(it);
            items.splice(index, 1);
        }

        // clamp helper
        private function clamp(v:Number, min:Number, max:Number):Number {
            if (v < min) return min;
            if (v > max) return max;
            return v;
        }

        // call to stop the game (optionally show gameover)
        private function gameOver():void {
            spawnTimer.stop();
            difficultyTimer.stop();
            removeEventListener(Event.ENTER_FRAME, onEnterFrame);
            stage.removeEventListener(KeyboardEvent.KEY_DOWN, onKeyDown);
            stage.removeEventListener(KeyboardEvent.KEY_UP, onKeyUp);
            stage.removeEventListener(Event.MOUSE_MOVE, onMouseMove);

            if (gameover_txt) {
                gameover_txt.visible = true;
                gameover_txt.text = "Game Over\nScore: " + score;
            }
        }
    }
}
// FallingItem.as (base class)
package {
    import flash.display.MovieClip;
    import flash.events.Event;

    public class FallingItem extends MovieClip {
        public var fallSpeed:Number = 3;
        public var rotationSpeed:Number = 0;
        public var isBomb:Boolean = false;

        public function FallingItem() {
            if (stage) addedToStage();
            else addEventListener(Event.ADDED_TO_STAGE, addedToStage);
        }

        private function addedToStage(e:Event = null):void {
            if (e) removeEventListener(Event.ADDED_TO_STAGE, addedToStage);
            // start any initialization
            addEventListener(Event.ENTER_FRAME, onEnterFrame);
        }

        protected function onEnterFrame(e:Event):void {
            update();
        }

        public function update():void {
            y += fallSpeed;
            rotation += rotationSpeed;
        }

        public function destroy():void {
            removeEventListener(Event.ENTER_FRAME, onEnterFrame);
            if (parent) parent.removeChild(this);
        }
    }
}
// FruitItem.as
package {
    public class FruitItem extends FallingItem {
        public function FruitItem() {
            super();
            isBomb = false;
            // optionally customize appearance or hit area here
        }
    }
}
// BombItem.as
package {
    public class BombItem extends FallingItem {
        public function BombItem() {
            super();
            isBomb = true;
            // bombs may fall a bit faster
            fallSpeed += 1.0;
            rotationSpeed *= 1.5;
        }
    }
}

  
