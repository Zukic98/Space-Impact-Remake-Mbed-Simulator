#include "mbed.h"
#include "stm32f413h_discovery_ts.h"
#include "stm32f413h_discovery_lcd.h"
#include <vector>
#include <cstdlib>
#include <list>
#include <map>
#include <string>

TS_StateTypeDef TS_State = { 0 };

enum Translation{UP,DOWN,LEFT,RIGHT};

int SCORE(0);

Ticker moveShipTicker;
Ticker spawnShipTicker;
Ticker allyFireTicker;
Ticker enemyFireTicker;

// stanje programa: menu, u igri, pauza
enum State {menu, inGame, pause,gameOver};
State state = menu;
bool triggerGame=false;//for game over

int abs (int i)
{
  return i < 0 ? -i : i;
}

void removeEnemyShip(int i);

class Artillery{
    protected:
        int power;
        int speed;
        Translation direction;
    public:
        Artillery(int power,int speed,Translation translation):power(power),speed(speed),direction(translation){}
        virtual void move()=0;
        virtual void show()const=0;
        virtual void unshow()const=0;
        virtual int getX()const=0;
        virtual int getY()const=0;
        int getPower()const{return power;}
        Translation getDirection()const{return direction;}
};

class CannonBall : public Artillery{
        int ballRadius;
        Point ball;
    public:
        CannonBall(int power,int speed,Point dot,int radius,Translation translation):Artillery(power,speed,translation){
            ballRadius=radius;
            ball=dot;
        }
        int getX()const override{return ball.X;}
        int getY()const override{return ball.Y;}
        void move(){if(direction==RIGHT)ball.X+=speed;else if(direction==LEFT)ball.X-=speed;}
        void show()const override{
            BSP_LCD_SetTextColor(LCD_COLOR_BLACK );
            BSP_LCD_FillCircle(ball.X, ball.Y, ballRadius);
        }
        void unshow()const override{
            BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
            BSP_LCD_FillCircle(ball.X, ball.Y, ballRadius);
        }
};

class Projectile : public Artillery{
        int elipseRadius;
        Point elipse;
    public:
        Projectile(int power,int speed,Point dot,int radius,Translation translation):Artillery(power,speed,translation){
            elipseRadius=radius;
            elipse=dot;
        }
        int getX()const override{return elipse.X;}
        int getY()const override{return elipse.Y;}
        void move() override{if(direction==RIGHT)elipse.X+=speed;else if(direction==LEFT)elipse.X-=speed;}
        void show()const{
            BSP_LCD_SetTextColor(LCD_COLOR_BLACK );
            BSP_LCD_FillEllipse(elipse.X, elipse.Y, elipseRadius+4,elipseRadius+1);
        }
        void unshow()const override{
            BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
            BSP_LCD_FillEllipse(elipse.X, elipse.Y, elipseRadius+4,elipseRadius+1);
        }
};

std::map<int,std::list<Artillery*> > allArtillery;

class BattleShip{
    protected:
        Point cornerUL;     // upper left corner of battleship
        Point cornerLR;     // lower right corner of battleship
        int health;
        int speed;
        int firePower;
    public:
        BattleShip(int h,int s,int fp, Point c);
        Point getCornerUL()const { return cornerUL; }
        Point getCornerLR()const { return cornerLR; }
        int getHealth()const{return health;}
        virtual void fire() =0;
        bool reduceHealth(int decrease){health-=decrease;if(health==0)return true;return false;};
        virtual void show()=0;
        virtual void unshow(){}
        virtual int move(Translation direction)=0;
        virtual bool isItShot(Point test)=0;
        virtual ~BattleShip(){};
};

BattleShip::BattleShip(int h,int s,int fp, Point c):health(h),speed(s),firePower(fp), cornerUL(c)
{
    // default values
    cornerLR.X = 0;
    cornerLR.Y = 0;
}

// global vector for enemyShips
std::vector<BattleShip*> enemyShips(0);

class AllyShip : public BattleShip{
    Point rectanglePoints[4];
    int rangeArtillery=3;
    void blink();
public:
    AllyShip(int h,int s,int fp, Point c):BattleShip(h,s,fp,c)
    {
        rectanglePoints[0].X=cornerUL.X + 3;
        rectanglePoints[0].Y=cornerUL.Y;
        rectanglePoints[1].X=cornerUL.X + 3;
        rectanglePoints[1].Y=cornerUL.Y + 9;
        rectanglePoints[2].X=cornerUL.X;
        rectanglePoints[2].Y=cornerUL.Y + 2;
        rectanglePoints[3].X=cornerUL.X;
        rectanglePoints[3].Y=cornerUL.Y + 20;
        cornerLR.X = cornerUL.X + 27;
        cornerLR.Y = cornerUL.Y + 25;
    }
    void show();
    int move(Translation direction);
    void fire();
    bool reduceHealth(int decrease);
    bool isItShot(Point artillery);     // checks if ally ship was shot
    bool wasHit();      // checks if ally ship was hit by another ship
    void unshow();
};

void AllyShip::fire(){
    Point aim;
    aim.X=getCornerLR().X+3;
    aim.Y=(getCornerLR().Y+getCornerUL().Y)/2;
    if(allArtillery[aim.Y].size()!=rangeArtillery){
        allArtillery[aim.Y].push_back(new CannonBall(firePower,3,aim,2,RIGHT));
    }
}

bool AllyShip::reduceHealth(int decrease)
{
    health -= decrease;
    BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
    int x = 71 + 13*health;
    int y = 7;
    int length = 13;
    BSP_LCD_FillRect(x, y, length, 8);
    if (health == 0)return true;
    return false;
}

void AllyShip::show(){
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    BSP_LCD_FillRect(rectanglePoints[0].X, rectanglePoints[0].Y, 5, 25);
    BSP_LCD_FillRect(rectanglePoints[1].X, rectanglePoints[1].Y, 25, 7);
    BSP_LCD_FillRect(rectanglePoints[2].X, rectanglePoints[2].Y, 14, 3);
    BSP_LCD_FillRect(rectanglePoints[3].X, rectanglePoints[3].Y, 14, 3);
}

void AllyShip::unshow(){
    BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
    BSP_LCD_FillRect(rectanglePoints[0].X, rectanglePoints[0].Y, 5, 25);
    BSP_LCD_FillRect(rectanglePoints[1].X, rectanglePoints[1].Y, 25, 7);
    BSP_LCD_FillRect(rectanglePoints[2].X, rectanglePoints[2].Y, 14, 3);
    BSP_LCD_FillRect(rectanglePoints[3].X, rectanglePoints[3].Y, 14, 3);
}

bool AllyShip::isItShot(Point artillery)
{
    if(artillery.Y>=getCornerUL().Y && artillery.Y<=getCornerLR().Y && artillery.X<=getCornerLR().X)
        return true;
    return false;
}

void AllyShip::blink(){
    for (int j = 0; j < 3; j++){
        show();
        wait_ms(50);
        unshow();
        wait_ms(50);
    }
    show();
}

bool AllyShip::wasHit()
{
    int allyLength = cornerLR.X - cornerUL.X;
    int allyHeight = cornerLR.Y - cornerUL.Y;
    for (int i = 0; i < enemyShips.size(); i++)
    {
        Point cul = enemyShips[i]->getCornerUL();
        if(abs(cornerUL.X - cul.X) <= allyLength  &&  abs(cornerUL.Y - cul.Y) <= allyHeight)
        {
            removeEnemyShip(i);
            i--;
            blink();
            return true;
        }
    }
    return false;
}

int AllyShip::move(Translation direction)
{
    // disappearing ship
    unshow();
    int k = speed;  // use k for speed of translation
    if(direction == UP)k = -speed;
    if(cornerUL.Y+k<149 && cornerUL.Y+k>19){
        rectanglePoints[0].Y+=k;
        rectanglePoints[1].Y+=k;
        rectanglePoints[2].Y+=k;
        rectanglePoints[3].Y+=k;
        cornerUL.Y += k;
        cornerLR.Y += k;
    }
    // draw of moving ship
    show();
    return 1;
}

int counterShips(0);

class EasyEnemyShip : public BattleShip{
        Point upTriangle[3];
        Point downTriangle[3];
        Point middleTriangle[3];
    public:
    EasyEnemyShip(int h,int s,int fp, Point c):BattleShip(h,s,fp,c)
    {
        
        //up triangle
        upTriangle[0].X=cornerUL.X + 10;
        upTriangle[0].Y=cornerUL.Y;
        upTriangle[1].X=cornerUL.X + 20;
        upTriangle[1].Y=cornerUL.Y + 5;
        upTriangle[2].X=cornerUL.X + 10;
        upTriangle[2].Y=cornerUL.Y + 10;
        //down triangle
        downTriangle[0].X=cornerUL.X + 10;
        downTriangle[0].Y=cornerUL.Y + 14;
        downTriangle[1].X=cornerUL.X + 20;
        downTriangle[1].Y=cornerUL.Y + 19;
        downTriangle[2].X=cornerUL.X + 10;
        downTriangle[2].Y=cornerUL.Y + 24;
        //middle triangle
        middleTriangle[0].X=cornerUL.X;
        middleTriangle[0].Y=cornerUL.Y + 12;
        middleTriangle[1].X=cornerUL.X + 15;
        middleTriangle[1].Y=cornerUL.Y + 4;
        middleTriangle[2].X=cornerUL.X + 15;
        middleTriangle[2].Y=cornerUL.Y + 19;
        // lower right corner
        cornerLR.X = c.X + 20;
        cornerLR.Y = c.Y + 24;
    }
    void show();
    int move(Translation direction);
    void fire(){};
    bool isItShot(Point artillery);
};

bool EasyEnemyShip::isItShot(Point artillery){
    if(artillery.Y>=getCornerUL().Y && artillery.Y<=getCornerLR().Y && artillery.X>=getCornerUL().X)
        return true;
    return false;
}

void EasyEnemyShip :: show(){
    BSP_LCD_FillPolygon(upTriangle,3);
    BSP_LCD_FillPolygon(downTriangle,3);
    BSP_LCD_FillPolygon(middleTriangle,3);
}

int EasyEnemyShip::move(Translation direction)
{
    // disappearing Ship
    BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
    show();
    // pomjeranje koordinata za "k" pixela
    int k = speed;  // mozemo staviti i speed ili koeficijent k
    if(direction == LEFT)k = -speed;
    if(middleTriangle[0].X+k>3){
        upTriangle[0].X+=k;
        upTriangle[1].X+=k;
        upTriangle[2].X+=k;
        //down triangle
        downTriangle[0].X+=k;
        downTriangle[1].X+=k;
        downTriangle[2].X+=k;
        //middle triangle
        middleTriangle[0].X+=k;
        middleTriangle[1].X+=k;
        middleTriangle[2].X+=k;
        // corners
        cornerUL.X += k;
        cornerLR.X += k;
    }
    else return -1;
        // return -1 if out of bounds
    // draw of moving ship
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    show();
    return true;
}

class MediumEnemyShip : public BattleShip{
        Point elipse[2];
        Point circle;
        int rangeArtillery=2;
    public:
    MediumEnemyShip(int h,int s,int fp, Point c):BattleShip(h,s,fp,c)
    {
        //elipse
        elipse[0].X=cornerUL.X + 16;
        elipse[0].Y=cornerUL.Y + 3;
        elipse[1].X=cornerUL.X + 16;
        elipse[1].Y=cornerUL.Y + 13;
        circle.X=cornerUL.X + 20;
        circle.Y=cornerUL.Y + 8;
        // lower right corner
        cornerLR.X = c.X + 32;
        cornerLR.Y = c.Y + 16;
    }
    void show();
    int move(Translation direction);
    bool isItShot(Point artillery);
    void fire();
};

void MediumEnemyShip::fire(){
    Point aim;
    aim.X=getCornerUL().X-3;
    aim.Y=getCornerUL().Y+2;
    if(allArtillery[aim.Y].size()!=rangeArtillery)
        allArtillery[aim.Y].push_front(new CannonBall(firePower,speed,aim,2,LEFT));
    aim.Y=getCornerLR().Y-2;
    if(allArtillery[aim.Y].size()!=rangeArtillery)
        allArtillery[aim.Y].push_front(new CannonBall(firePower,speed,aim,2,LEFT));
}

void MediumEnemyShip::show(){
    BSP_LCD_FillEllipse(elipse[0].X, elipse[0].Y, 10, 3);
    BSP_LCD_FillEllipse(elipse[1].X, elipse[1].Y, 10, 3);
    BSP_LCD_FillCircle(circle.X, circle.Y, 4);
}

int MediumEnemyShip::move(Translation direction)
{
    // unshow ship
    BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
    show();
    // for value of speed move 
    int k = speed;  
    if(direction == LEFT)k = -speed;
    if(cornerUL.X+k>3){
        elipse[0].X+=k;
        elipse[1].X+=k;
        circle.X+=k;
        // corners
        cornerUL.X += k;
        cornerLR.X += k;
    }
    else
    {
        // return -1 if out of bounds
        return -1;
    }
    // draw of moving ship
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    show();
    return 1;
}

bool MediumEnemyShip::isItShot(Point artillery){
    if(artillery.Y>=getCornerUL().Y && artillery.Y<=getCornerLR().Y && artillery.X>=getCornerUL().X)
        return true;
    return false;
}

class HardEnemyShip : public BattleShip{
        Point rectangle[5];
        Point upTriangle[3];
        Point downTriangle[3];
        int rangeArtillery=1;
    public:
    HardEnemyShip(int h,int s,int fp, Point c):BattleShip(h,s,fp,c)
    {
        
        //up rectangle
        rectangle[0].X=cornerUL.X + 10;
        rectangle[0].Y=cornerUL.Y;
        //down rectangle
        rectangle[1].X=cornerUL.X + 10;
        rectangle[1].Y=cornerUL.Y + 15;
        //right rectangle
        rectangle[2].X=cornerUL.X + 17;
        rectangle[2].Y=cornerUL.Y;
        //up bottom rectangle
        rectangle[3].X=cornerUL.X + 13;
        rectangle[3].Y=cornerUL.Y + 5;
        //down bottom rectangle
        rectangle[4].X=cornerUL.X + 13;
        rectangle[4].Y=cornerUL.Y + 12;
        //up Triangle
        upTriangle[0].X=cornerUL.X + 10;
        upTriangle[0].Y=cornerUL.Y;
        upTriangle[1].X=cornerUL.X + 10;
        upTriangle[1].Y=cornerUL.Y + 5;
        upTriangle[2].X=cornerUL.X;
        upTriangle[2].Y=cornerUL.Y;
        //down Triangle
        downTriangle[0].X=cornerUL.X + 10;
        downTriangle[0].Y=cornerUL.Y + 15;
        downTriangle[1].X=cornerUL.X + 10;
        downTriangle[1].Y= cornerUL.Y + 20;
        downTriangle[2].X=cornerUL.X;
        downTriangle[2].Y=cornerUL.Y + 20;
        // lower right corner
        cornerLR.X = c.X + 20;
        cornerLR.Y = c.Y + 24;
    }
    void show();
    int move(Translation direction);
    bool isItShot(Point artillery);
    void fire();
};

void HardEnemyShip::fire(){
    Point aim;
    aim.X=getCornerUL().X-1;
    aim.Y=getCornerUL().Y+1;
    if(allArtillery[aim.Y].size()!=rangeArtillery)
        allArtillery[aim.Y].push_front(new CannonBall(firePower,7,aim,2,LEFT));
    aim.Y=getCornerLR().Y-1;
    if(allArtillery[aim.Y].size()!=rangeArtillery)
        allArtillery[aim.Y].push_front(new CannonBall(firePower,7,aim,2,LEFT));
    aim.Y=(getCornerLR().Y+getCornerUL().Y)/2;
    if(allArtillery[aim.Y].size()!=rangeArtillery)
        allArtillery[aim.Y].push_front(new CannonBall(firePower,7,aim,2,LEFT));
}

bool HardEnemyShip::isItShot(Point artillery){
    if(artillery.Y>=getCornerUL().Y && artillery.Y<=getCornerLR().Y && artillery.X>=getCornerUL().X)
        return true;
    return false;
}

void HardEnemyShip::show(){
    BSP_LCD_FillRect(rectangle[0].X, rectangle[0].Y, 13, 5);
    BSP_LCD_FillRect(rectangle[1].X, rectangle[1].Y, 13, 5);
    BSP_LCD_FillRect(rectangle[2].X, rectangle[2].Y, 5, 20);
    BSP_LCD_FillRect(rectangle[3].X, rectangle[3].Y, 15, 3);
    BSP_LCD_FillRect(rectangle[4].X, rectangle[4].Y, 15, 3);
    BSP_LCD_FillPolygon(upTriangle,3);
    BSP_LCD_FillPolygon(downTriangle,3);
}

int HardEnemyShip::move(Translation direction)
{
    // unshow
    BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
    show();
    // move for value of speed
    int k = speed;  
    if(direction == LEFT)k = -speed;
    if(upTriangle[2].X+k>3){
        //up rectangle
        rectangle[0].X+=k;
        //down rectangle
        rectangle[1].X+=k;
        //right rectangle
        rectangle[2].X+=k;
        //up bottom rectangle
        rectangle[3].X+=k;
        //down bottom rectangle
        rectangle[4].X+=k;
        //up Triangle
        upTriangle[0].X+=k;
        upTriangle[1].X+=k;
        upTriangle[2].X+=k;
        //down Triangle
        downTriangle[0].X+=k;
        downTriangle[1].X+=k;
        downTriangle[2].X+=k;
        // corners
        cornerUL.X += k;
        cornerLR.X += k;
    }
    else
    {
        // return -1 if out of bounds
        return -1;
    }
    // draw of moving ship
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    show();
    return 1;
}

class Boss : public BattleShip{
        Point crown1[3];
        Point crown2[3];
        Point crown3[3];
        Point mainFrame;
        Point eyeLeft;
        Point eyeRight;
        Point firstBigCannon;
        Point secondBigCannon;
        Point thirdBigCannon;
        Point firstSmallCannon;
        Point secondSmallCannon;
        void makeCrown();
        void makeMainFrame();
        void makeSmallCannons();
        void makeBigCannons();
        int rangeArtillerySmall=3;
        void fireCannon1();
        void fireCannon2();
        void fireBigCannon1();
        void fireBigCannon2();
        void fireBigCannon3();
    public:
        Boss(int h, int s, int fp, Point c) : BattleShip(h, s, fp, c)
        {
            cornerLR.X = c.X + 60;
            cornerLR.Y = c.Y + 110;
        }
        void fire();
        void show();
        int move(Translation direction);
        bool isItShot(Point artillery);
        void unshow();
};

void Boss::fire(){
    srand(time(NULL));
    int dice=1+rand() % 5;
    switch(dice){
        case 1:
        fireCannon1();
        break;
        case 2:
        fireCannon2();
        break;
        case 3:
        fireBigCannon1();
        break;
        case 4:
        fireBigCannon2();
        break;
        case 5:
        fireBigCannon3();
        break;
    }
}

void Boss::fireCannon1(){
    Point aim;
    aim.X=firstSmallCannon.X-3;
    aim.Y=firstSmallCannon.Y+2;
    if(allArtillery[aim.Y].size()!=rangeArtillerySmall)
        allArtillery[aim.Y].push_front(new CannonBall(firePower,7,aim,2,LEFT));
}

void Boss::fireCannon2(){
    Point aim;
    aim.X=secondSmallCannon.X-3;
    aim.Y=secondSmallCannon.Y+2;
    if(allArtillery[aim.Y].size()!=rangeArtillerySmall)
        allArtillery[aim.Y].push_front(new CannonBall(firePower,7,aim,2,LEFT));
}


void Boss::fireBigCannon1(){
    Point aim;
    aim.X=firstBigCannon.X-10;
    aim.Y=firstBigCannon.Y+2;
    if(allArtillery[aim.Y].size()!=rangeArtillerySmall)
        allArtillery[aim.Y].push_front(new Projectile(firePower,7,aim,1,LEFT));
}


void Boss::fireBigCannon2(){
    Point aim;
    aim.X=secondBigCannon.X-10;
    aim.Y=secondBigCannon.Y+2;
    if(allArtillery[aim.Y].size()!=rangeArtillerySmall)
        allArtillery[aim.Y].push_front(new Projectile(1,7,aim,1,LEFT));
}


void Boss::fireBigCannon3(){
    Point aim;
    aim.X=thirdBigCannon.X-10;
    aim.Y=thirdBigCannon.Y+2;
    if(allArtillery[aim.Y].size()!=rangeArtillerySmall)
        allArtillery[aim.Y].push_front(new Projectile(1,7,aim,1,LEFT));
}

bool Boss::isItShot(Point artillery){
    if(artillery.Y>=getCornerUL().Y && artillery.Y<=getCornerLR().Y && artillery.X>=getCornerUL().X)
        return true;
    return false;
}

void Boss::unshow(){
    BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
    BSP_LCD_FillRect(180, 50, 30, 70);
}

void Boss::makeCrown(){
    //leftTriangle
    crown1[0].X = cornerUL.X;
    crown1[0].Y = cornerUL.Y;
    crown1[1].X = cornerUL.X;
    crown1[1].Y = cornerUL.Y + 20;
    crown1[2].X = cornerUL.X + 20;
    crown1[2].Y = cornerUL.Y + 20;
    //middleTriangle
    crown2[0].X = cornerUL.X + 30;
    crown2[0].Y = cornerUL.Y;
    crown2[1].X = cornerUL.X + 20;
    crown2[1].Y = cornerUL.Y + 20;
    crown2[2].X = cornerUL.X + 40;
    crown2[2].Y = cornerUL.Y + 20;
    //rightTriangle
    crown3[0].X = cornerUL.X + 60;
    crown3[0].Y = cornerUL.Y;
    crown3[1].X = cornerUL.X + 40;
    crown3[1].Y = cornerUL.Y + 20;
    crown3[2].X = cornerUL.X + 60;
    crown3[2].Y = cornerUL.Y + 20;
    BSP_LCD_FillPolygon(crown1, 3);
    BSP_LCD_FillPolygon(crown2, 3);
    BSP_LCD_FillPolygon(crown3, 3);
}

void Boss::makeMainFrame(){
    mainFrame.X = cornerUL.X;
    mainFrame.Y = cornerUL.Y + 20;
    BSP_LCD_FillRect(mainFrame.X, mainFrame.Y, 61, 90);
    eyeLeft.X = cornerUL.X + 10;
    eyeLeft.Y = cornerUL.Y + 40;
    eyeRight.X = cornerUL.X + 30;
    eyeRight.Y = cornerUL.Y + 40;
    BSP_LCD_SetTextColor(LCD_COLOR_RED);
    BSP_LCD_FillRect(eyeLeft.X, eyeLeft.Y, 10, 10);
    BSP_LCD_FillRect(eyeRight.X, eyeRight.Y, 10, 10);
}

void Boss::makeSmallCannons(){
    firstSmallCannon.X = cornerUL.X - 5;
    firstSmallCannon.Y = cornerUL.Y + 50;
    secondSmallCannon.X = cornerUL.X - 5;
    secondSmallCannon.Y = cornerUL.Y + 80;
    BSP_LCD_FillRect(firstSmallCannon.X, firstSmallCannon.Y, 5, 5);
    BSP_LCD_FillRect(secondSmallCannon.X, secondSmallCannon.Y, 5, 5);
}

void Boss::makeBigCannons(){
    firstBigCannon.X = cornerUL.X - 25;
    firstBigCannon.Y = cornerUL.Y + 30;
    secondBigCannon.X = cornerUL.X - 25;
    secondBigCannon.Y = cornerUL.Y + 65;
    thirdBigCannon.X = cornerUL.X - 25;
    thirdBigCannon.Y = cornerUL.Y + 95;
    
    BSP_LCD_FillRect(firstBigCannon.X, firstBigCannon.Y, 25, 7);
    BSP_LCD_FillRect(secondBigCannon.X, secondBigCannon.Y, 25, 7);
    BSP_LCD_FillRect(thirdBigCannon.X, thirdBigCannon.Y, 25, 7);
}

void Boss::show(){
    makeCrown();
    makeMainFrame();
    BSP_LCD_SetTextColor(LCD_COLOR_BROWN);
    makeSmallCannons();
    makeBigCannons();
}

int Boss::move(Translation direction)
{
    while(cornerUL.X > 160)
    {
        BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
        makeCrown();
        BSP_LCD_FillRect(cornerUL.X, cornerUL.Y, cornerLR.X - cornerUL.X + 3, cornerLR.Y - cornerUL.Y);
        makeBigCannons();
        makeSmallCannons();
        BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
        show();
        cornerUL.X -= 2;
        cornerLR.X -= 2;
        wait_us(1);
    }
    return 0;
}

// moves all enemy ships
void moveEnemyShips()
{
    for(int i = 0; i < enemyShips.size(); i++)
    {
       
        if (enemyShips[i]->move(LEFT) == -1)
        {
            removeEnemyShip(i);
            i--;
        }
    }
    // make correction of left right side od display
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    BSP_LCD_DrawVLine(BSP_LCD_GetXSize()-1, 0, BSP_LCD_GetYSize());
    BSP_LCD_DrawVLine(BSP_LCD_GetXSize()-2, 0, BSP_LCD_GetYSize());
    BSP_LCD_DrawVLine(BSP_LCD_GetXSize()-3, 0, BSP_LCD_GetYSize());
}

// removes enemy ships
void removeEnemyShip(int i)
{
    BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
    enemyShips[i]->show();
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    if(dynamic_cast<Boss*>(enemyShips[i]) && enemyShips[i]->getHealth()<=0)triggerGame=true;
    delete enemyShips[i];
    enemyShips.erase(enemyShips.begin()+i);
}

std::vector<int> enemyLayout(6,0);

void spawnEnemyShips()
{
    // check which position is not available
    bool isFree = false;
    for (int i = 0; i < enemyLayout.size(); i++)
    {
        if (enemyLayout[i] == 0)
        {
            isFree = true;
            break;
        }
    }
    // if all postions are taken,clear
    if (!isFree)
    {
        for (int i = 0; i < enemyLayout.size(); i++)
            enemyLayout[i] = 0;
        return;       
    }
    // generate random six position
    srand(time(NULL));
    int randomPosition=rand() % 6;
    while(enemyLayout[randomPosition] == 1)
    {
        randomPosition =rand() % 6;
    }
    enemyLayout[randomPosition] = 1;
    // define position of enemyShip
    Point position;
    position.X = 230;
    position.Y = 19 + randomPosition*26;
    if (counterShips<7)
    {
        enemyShips.push_back(new EasyEnemyShip(3,7,1,position));
        counterShips++;
    }
    else if (counterShips<14)
    {
        enemyShips.push_back(new MediumEnemyShip(5,5,1,position));
        counterShips++;
    }
    else if (counterShips<20)
    {
        enemyShips.push_back(new HardEnemyShip(7,3,1,position));
        counterShips++;
    }else if(enemyShips.size()<=1){
        position.X = 225;
        position.Y = 35;
        enemyShips.push_back(new Boss(15,3,1, position));
        spawnShipTicker.detach();
    }
}

void getMainFrame(){
    //setting frame for game
    BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
    BSP_LCD_FillRect(0,0,BSP_LCD_GetXSize(),BSP_LCD_GetYSize());
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    BSP_LCD_SetBackColor(LCD_COLOR_DARKGREEN);
    BSP_LCD_SetFont(&Font16);
    BSP_LCD_DisplayStringAt(0, 80, (uint8_t *)"PLAY", CENTER_MODE);
    BSP_LCD_DisplayStringAt(0, 50, (uint8_t *)"Space Impact", CENTER_MODE);
    // make background of bottom gray
    BSP_LCD_SetTextColor(LCD_COLOR_LIGHTGRAY);
    BSP_LCD_FillRect(3, BSP_LCD_GetYSize()-65, BSP_LCD_GetXSize()-3, BSP_LCD_GetYSize()-3);
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    //bold lines for frame bottom of corner
    BSP_LCD_DrawHLine(3, BSP_LCD_GetYSize()-65, BSP_LCD_GetXSize()-3);
    BSP_LCD_DrawHLine(3, BSP_LCD_GetYSize()-64, BSP_LCD_GetXSize()-3);
    BSP_LCD_DrawHLine(3, BSP_LCD_GetYSize()-63, BSP_LCD_GetXSize()-3);
    //left vertical line
    BSP_LCD_DrawVLine(0, 0, BSP_LCD_GetYSize());
    BSP_LCD_DrawVLine(1, 0, BSP_LCD_GetYSize());
    BSP_LCD_DrawVLine(2, 0, BSP_LCD_GetYSize());
    //right vertical line
    BSP_LCD_DrawVLine(BSP_LCD_GetXSize()-1, 0, BSP_LCD_GetYSize());
    BSP_LCD_DrawVLine(BSP_LCD_GetXSize()-2, 0, BSP_LCD_GetYSize());
    BSP_LCD_DrawVLine(BSP_LCD_GetXSize()-3, 0, BSP_LCD_GetYSize());
    //up horizontal line
    BSP_LCD_DrawHLine(0, 0, BSP_LCD_GetXSize());
    BSP_LCD_DrawHLine(0, 1, BSP_LCD_GetXSize());
    BSP_LCD_DrawHLine(0, 2, BSP_LCD_GetXSize());
    //bottom horizontal line,down
    BSP_LCD_DrawHLine(0, BSP_LCD_GetYSize()-3, BSP_LCD_GetXSize());
    BSP_LCD_DrawHLine(0, BSP_LCD_GetYSize()-1, BSP_LCD_GetXSize());
    BSP_LCD_DrawHLine(0, BSP_LCD_GetYSize()-2, BSP_LCD_GetXSize());
    //arrow up
    BSP_LCD_DrawRect(15, 182, 60, 25);
    Point tackeTrougla[4];
    tackeTrougla[0].X=45;
    tackeTrougla[0].Y=187;
    tackeTrougla[1].X=25;
    tackeTrougla[1].Y=202;
    tackeTrougla[2].X=65;
    tackeTrougla[2].Y=202;
    BSP_LCD_FillPolygon(tackeTrougla,3);
    //arrow down
    BSP_LCD_DrawRect(15, 207, 60, 25);
    tackeTrougla[0].X=25;
    tackeTrougla[0].Y=212;
    tackeTrougla[1].X=65;
    tackeTrougla[1].Y=212;
    tackeTrougla[2].X=45;
    tackeTrougla[2].Y=227;
    BSP_LCD_FillPolygon(tackeTrougla,3);
    //fire button
        // button border
    BSP_LCD_DrawCircle(102, 207, 20);
    BSP_LCD_DrawCircle(102, 207, 21);
    BSP_LCD_DrawCircle(102, 207, 22);
        // red bullseye
    BSP_LCD_SetTextColor(LCD_COLOR_RED);
    BSP_LCD_DrawCircle(102, 207, 7);
    BSP_LCD_DrawCircle(102, 207, 8);
    BSP_LCD_DrawCircle(102, 207, 9);
    BSP_LCD_FillRect(101,195,3,9);
    BSP_LCD_FillRect(101,210,3,9);
    BSP_LCD_FillRect(90,206,9,2);
    BSP_LCD_FillRect(106,206,9,2);
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    //play-pause
    BSP_LCD_DrawCircle(190, 207, 20);
    BSP_LCD_DrawCircle(190, 207, 21);
    BSP_LCD_DrawCircle(190, 207, 22);
    BSP_LCD_FillRect(183,197,5,20);
    BSP_LCD_FillRect(193,197,5,20);
    //status bar
    BSP_LCD_SetBackColor(LCD_COLOR_LIGHTBLUE);
    BSP_LCD_DisplayStringAt(50, 7, (uint8_t *)"Score: 0", CENTER_MODE);
    BSP_LCD_SetBackColor(LCD_COLOR_LIGHTBLUE);
    //BSP_LCD_DisplayStringAt(20, 10, (uint8_t *)"Health: 3", LEFT_MODE);
    // health bar (srca umjesto broja)
    BSP_LCD_DisplayStringAt(20, 7, (uint8_t *)"Health:", LEFT_MODE);
    int a = 73;
    int b = 7;
    for(int i = 0; i < 3; i++)
    {
        BSP_LCD_SetTextColor(LCD_COLOR_LIGHTRED);
        BSP_LCD_DrawHLine (a, b, 2);
        BSP_LCD_DrawHLine (a+5, b, 2);
        BSP_LCD_DrawHLine (a-1, b+1, 4);
        BSP_LCD_DrawHLine (a+4, b+1, 4);
        BSP_LCD_DrawHLine (a-2, b+2, 11);
        BSP_LCD_DrawHLine (a-2, b+3, 11);
        BSP_LCD_DrawHLine (a-1, b+4, 9);
        BSP_LCD_DrawHLine (a, b+5, 7);
        BSP_LCD_DrawHLine (a+1, b+6, 5);
        BSP_LCD_DrawHLine (a+2, b+7, 3);
        BSP_LCD_DrawHLine (a+3, b+8, 1);
        BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
        a+=13;
    }
    tackeTrougla[0].X = 12;
    tackeTrougla[0].Y = 80;
    AllyShip alliance(3,3,3,tackeTrougla[0]);
    alliance.show();
}

Point backPoint(){
    Point p;p.X=0;p.Y=0;
    return p;
}
//we make this because some problems with simulator about Point as global variable
AllyShip alliance(3,3,3,backPoint());

int countDigit(){
    auto tmp(SCORE);
    int counter(0);
    if(!SCORE)return 1;
    while(tmp!=0){
        tmp/=10;
        counter++;
    }
    return counter;
}

void resetScore(){
    char *c(new char [9+countDigit()]);
    std::string score("Score: "+std::to_string(SCORE));
    strcpy(c,score.c_str());
    BSP_LCD_SetBackColor(LCD_COLOR_LIGHTBLUE);
    BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
    BSP_LCD_DisplayStringAt(50, 7, (uint8_t *)c, CENTER_MODE);
    BSP_LCD_SetBackColor(LCD_COLOR_LIGHTBLUE);
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    BSP_LCD_DisplayStringAt(50, 7, (uint8_t *)c, CENTER_MODE);
    delete[] c;
}

void updateArtillery(){
    for(std::map<int,std::list<Artillery*> >::iterator itAlly(allArtillery.begin());itAlly!=allArtillery.end();itAlly++){
        for(std::list<Artillery*>::iterator walker(itAlly->second.begin());walker!=itAlly->second.end();){
            bool trigger(false);
            (*walker)->show();
            (*walker)->unshow();
            (*walker)->move();
            (*walker)->show();
            Point testPoint;
            testPoint.X=(*walker)->getX();
            testPoint.Y=(*walker)->getY();
            auto tmp=walker;
            tmp++;
            if(alliance.isItShot(testPoint)){
                if(alliance.reduceHealth((*walker)->getPower()))triggerGame=true;
                (*walker)->unshow();
                auto tmp=walker;
                walker++;
                trigger=true;
                delete *tmp;
                allArtillery[testPoint.Y].erase(tmp);
            }
            if(triggerGame)break;
            //check cannonBall to cannonBall->delete two cannonballs
            if(tmp!=itAlly->second.end() && (*walker)->getDirection()!=(*tmp)->getDirection() &&
                (*walker)->getX()<=(*tmp)->getX()){
                auto tmp1=walker;
                walker++;
                delete *tmp1;
                allArtillery[testPoint.Y].erase(tmp1);
                walker++;
                delete *tmp;
                allArtillery[testPoint.Y].erase(tmp);
                trigger=true;
            }//check cannonBall to enemyShip
            for(std::vector<BattleShip*>::iterator throughEnemyShips(enemyShips.begin());
                throughEnemyShips!=enemyShips.end();throughEnemyShips++){
                if((*throughEnemyShips)->isItShot(testPoint)){
                    if(dynamic_cast<EasyEnemyShip*>(*throughEnemyShips))SCORE+=50;
                    else if(dynamic_cast<MediumEnemyShip*>(*throughEnemyShips))SCORE+=100;
                    else if(dynamic_cast<HardEnemyShip*>(*throughEnemyShips))SCORE+=150;
                    else if(dynamic_cast<Boss*>(*throughEnemyShips))SCORE+=200;
                    resetScore();
                    if((*throughEnemyShips)->reduceHealth((*walker)->getPower()))
                       removeEnemyShip(throughEnemyShips-enemyShips.begin());
                    (*walker)->unshow();
                    delete *walker;
                    walker++;
                    trigger=true;
                    allArtillery[testPoint.Y].pop_front();
                    break;//just to speed
                }
            }
            //check the wall
            if(testPoint.X>=BSP_LCD_GetXSize() || testPoint.X<=0){
                (*walker)->unshow();
                delete *walker;
                auto tmp=walker;
                walker++;
                allArtillery[testPoint.Y].erase(tmp);
                trigger=true;
                BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
                BSP_LCD_DrawVLine(BSP_LCD_GetXSize()-1, 0, BSP_LCD_GetYSize());
                BSP_LCD_DrawVLine(BSP_LCD_GetXSize()-2, 0, BSP_LCD_GetYSize());
                BSP_LCD_DrawVLine(BSP_LCD_GetXSize()-3, 0, BSP_LCD_GetYSize());
                BSP_LCD_DrawVLine(0, 0, BSP_LCD_GetYSize());
                BSP_LCD_DrawVLine(1, 0, BSP_LCD_GetYSize());
                BSP_LCD_DrawVLine(2, 0, BSP_LCD_GetYSize());
            }
            if(!trigger){walker++;trigger=false;}
        }
        if(triggerGame)break;
    }
}

void randomFire(){
    srand(time(NULL));
    int randomNumber=rand()%enemyShips.size();
    if(!dynamic_cast<EasyEnemyShip*>(enemyShips[randomNumber]))enemyShips[randomNumber]->fire();
}

void shutDownTickers(){
    moveShipTicker.detach();
    spawnShipTicker.detach();
    allyFireTicker.detach();
    enemyFireTicker.detach();
}

void turnOnTickers(){
    moveShipTicker.attach(&moveEnemyShips, 0.1);
    spawnShipTicker.attach(&spawnEnemyShips, 2);
    allyFireTicker.attach(&updateArtillery,0.01);
    enemyFireTicker.attach(&randomFire,2);
}

void startGame()
{
    state = inGame;
    BSP_LCD_SetTextColor(LCD_COLOR_LIGHTBLUE);
    BSP_LCD_FillRect(49, 50, 132,50);
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    turnOnTickers();
}

void fire(AllyShip ally) {
    ally.fire();
}

void pauseGame()
{
    if(state == inGame)
    {
        shutDownTickers();
        state = pause;
        BSP_LCD_SetTextColor(LCD_COLOR_LIGHTGRAY);
        BSP_LCD_FillCircle(190, 207, 19);
        BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
        Point tackeTrougla[3];
        tackeTrougla[0].X=182;
        tackeTrougla[0].Y=192;
        tackeTrougla[1].X=182;
        tackeTrougla[1].Y=222;
        tackeTrougla[2].X=205;
        tackeTrougla[2].Y=207;
        BSP_LCD_FillPolygon(tackeTrougla,3);
        wait_ms(500);
    }
    else if (state == pause)
    {
        BSP_LCD_SetTextColor(LCD_COLOR_LIGHTGRAY);
        BSP_LCD_FillCircle(190, 207, 19);
        BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
        BSP_LCD_FillRect(183,197,5,20);
        BSP_LCD_FillRect(193,197,5,20);
        wait_ms(500);
        state = inGame;
        turnOnTickers();
    }
}

bool clickPlay(int x1, int y1) { return x1 >= 92 && x1 <= 137 && y1 >= 79 && y1 <= 95; }
bool clickUp(int x1, int y1) { return x1 >= 16 && x1 <= 76 && y1 >= 182 && y1 <= 208; }
bool clickDown(int x1, int y1) { return x1 >= 16 && x1 <= 76 && y1 > 208 && y1 <= 232; }
bool clickFire(int x1, int y1)
{
    int a = 102 - x1;
    int b = 207 - y1;
    return sqrt(a*a + b*b) <= 22;
}
bool clickPause(int x1, int y1)
{
    int a = 191 - x1;
    int b = 207 - y1;
    return sqrt(a*a + b*b) <= 23;
}

void deleteAllShips(){
     for(auto it(enemyShips.begin());it!=enemyShips.end();){
        auto tmp(it);
        it++;
        delete *tmp;
     }
    enemyShips.resize(0);
}

void deleteAllArtillery(){
    for(auto itAlly(allArtillery.begin());itAlly!=allArtillery.end();itAlly++){
        for(auto walker(itAlly->second.begin());walker!=itAlly->second.end();){
            auto tmp=walker;
            walker++;
            delete *tmp;
        }
        itAlly->second.clear();
    }
}

void showGameOverFrame(){
    BSP_LCD_SetTextColor(LCD_COLOR_BLACK);
    BSP_LCD_SetBackColor(LCD_COLOR_DARKGREEN);
    BSP_LCD_SetFont(&Font16);
    BSP_LCD_DisplayStringAt(0, 105, (uint8_t *)"PLAY AGAIN", CENTER_MODE);
    std::string score("Game over!");
    char *c(new char [score.length()+1]);
    strcpy(c,score.c_str());
    BSP_LCD_DisplayStringAt(0, 65, (uint8_t *)c, CENTER_MODE);
    score="*Your score is "+std::to_string(SCORE)+"*";
    delete[] c;
    c=new char [score.length()+1];
    strcpy(c,score.c_str());
    BSP_LCD_DisplayStringAt(0, 85, (uint8_t *)c, CENTER_MODE);
    delete[] c;
}

void intro(){
        BSP_LCD_Clear(LCD_COLOR_WHITE);
    getMainFrame();
    Point dot;
    dot.X = 12;
    dot.Y = 80;
    alliance=AllyShip(3,4,1,dot);
    // initialization of alliance
    // first element is blinking so we make dummy ship
    dot.X = 300;
    dot.Y=300;
    BattleShip *dummy= new EasyEnemyShip(0,0,0,dot);
    enemyShips.push_back(dummy);
}

void endGame()
{
    if(state == inGame)
    {
        state=gameOver;
        shutDownTickers();
        deleteAllShips();
        deleteAllArtillery();
        showGameOverFrame();
    }
    else if (state == gameOver)
    {
        SCORE=0;
        counterShips=0;
        intro();
        startGame();
    }
}

bool clickPlayAgain(int x1, int y1) { return x1 >= 60 && x1 <= 180 && y1 >= 105 && y1 <= 121; }

int main() {

    BSP_LCD_Init();

    /* Touchscreen initialization */
    if (BSP_TS_Init(BSP_LCD_GetXSize(), BSP_LCD_GetYSize()) == TS_ERROR) {
        printf("BSP_TS_Init error\n");
    }
    /* Clear the LCD */
    intro();
    while (1) {
        BSP_TS_GetState(&TS_State);
        if(TS_State.touchDetected) {
            /* One or dual touch have been detected          */
            /* Get X and Y position of the first touch post calibrated */
            uint16_t x1 = TS_State.touchX[0];
            uint16_t y1 = TS_State.touchY[0];
            switch(state)
            {
                case menu:
                    if(clickPlay(x1, y1))
                        startGame();
                    break;
                case inGame:
                    if(clickUp(x1,y1))
                        alliance.move(UP);
                    else if(clickDown(x1,y1))
                        alliance.move(DOWN);
                    else if (clickFire(x1, y1))
                        fire(alliance);
                    else if (clickPause(x1, y1))
                        pauseGame();
                    break;
                case pause:
                    if (clickPause(x1, y1))
                        pauseGame();
                    break;
                case gameOver:
                    if (clickPlayAgain(x1, y1))
                        endGame();
                    break;
            }
        }
        if (state == inGame && alliance.wasHit())
                if(alliance.reduceHealth(1))
                    triggerGame=true;
        if(triggerGame){triggerGame=false;endGame();}
        wait_ms(10);
    }
}
