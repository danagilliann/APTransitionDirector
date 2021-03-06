# APTransitionDirector
APTransitionDirector

# Example Usage
You could use director in one of this ways

* Basical Usage (for interactive and staic trasitions)
* category (APTransitions) & APTransitionProtocol (for static transitions)
* category (APTransitions) & blocks (for static transitions) - not implemented yet
* category (APRuleInteractiveTransitions) & APTransitionRule (for interactive transitions)
* category (APInteractiveTransitions) (for interactive transitions, the same as pervious but faster to implement) - not inplemented yet
 and more...


## Basical Usage

The main class is APTransitionDirector - its responds of all trasition protocols that iOS have:
``` objective-c
<UIViewControllerAnimatedTransitioning,UINavigationControllerDelegate,UIViewControllerInteractiveTransitioning,UITabBarControllerDelegate,UIViewControllerTransitioningDelegate>
```
its super fast to create u own transition, even with basical usage(see the category usage below):
``` objective-c
APTransitionDirector * director=[[APTransitionDirector alloc]init];
director.animBlock=^(APTransitionDirector * director ,void(^comlitBlock)() ){

};
self.navigationController.delegate=director;
[self.navigationController pushViewController:[[BUViewController alloc] init] animated:YES]
self.navigationController.delegate=nil;
```
Now you have the animation block ,where u could use any animations that u want,to get all needed views use the
APFastAcces category of APTransitionDirector. Lets Add Some Cool Animations:

``` objective-c
//BURootViewController.m
APTransitionDirector * director=[[APTransitionDirector alloc]init];
director.animDuration=0.5; //animation duration for director
director.animBlock=^(APTransitionDirector * director ,void(^comlitBlock)() ){
    //getting all needed views
    UIView* toView = [director toView];
    UIView* fromView= [director fromView];
    UIView *  containerView=  [director containerView];
   //presetup
    [containerView insertSubview:toView aboveSubview:fromView];
    [toView setFrame:CGRectMake(containerView.frame.size.width, toView.frame.origin.y, toView.frame.size.width, toView.frame.size.height)];
    fromView.transform=CGAffineTransformMakeScale(1.0, 1.0);
    //animation block
    [UIView animateWithDuration:director.animDuration delay:0 usingSpringWithDamping:0.8 initialSpringVelocity:0.1 options:UIViewAnimationOptionCurveEaseIn
    animations:^{
        [toView setCenter:CGPointMake(containerView.frame.size.width/2, toView.center.y)];
        fromView.transform=CGAffineTransformMakeScale(0.9, 0.9);
        } completion:^(BOOL finished) {
            fromView.transform=CGAffineTransformMakeScale(1.0, 1.0);
  //dont forget to call the complition block
            comlitBlock();
    }];

};
self.navigationController.delegate=director;
[self.navigationController pushViewController:viewController animated:YES]
self.navigationController.delegate=nil;
```
Pls note: When u use non interactive animations dont forget to call the complitBlock
Also u could use `@property (nonatomic)float animDuration;` to make all things clear.

Ok,Great! But we could make it better by adding some interactive transitions.
Let add EdgePanGesture to our navigationController:
``` objective-c
//BURootViewController.m
UIScreenEdgePanGestureRecognizer * panGesture = [[UIScreenEdgePanGestureRecognizer alloc]initWithTarget:self action:@selector(screenPan:)];
[panGesture setDelegate:self];
panGesture.edges=UIRectEdgeLeft;
self.navigationController.interactivePopGestureRecognizer.enabled = NO;
[self.navigationController.view addGestureRecognizer:panGesture];
``` 
and the screenPan Method:
``` objective-c
//BURootViewController.m
- (void)screenPan:(UIGestureRecognizer*)pan {
    CGPoint location = [pan locationInView:self.view];
    static CGPoint firstTouch;
    static float fullDistance=0;
    static APTransitionDirector * animDirector=nil;

    switch (pan.state) {
        case UIGestureRecognizerStateBegan:
        {

            fullDistance=self.navigationController.view.frame.size.width;
            firstTouch=location;

            animDirector=[[APTransitionDirector alloc]init];
            animDirector.interactive=YES;
            animDirector.animDuration=0.33;
            [self.navigationController setDelegate:animDirector];

            animDirector.animBlock=^(APTransitionDirector * director,void(^comlitBlock)() ){

                UIView* toView = [director toView];
                UIView* fromView= [director fromView];
                UIView *  containerView=  [director containerView];

                [containerView insertSubview:toView belowSubview:fromView];
                toView.transform=CGAffineTransformMakeScale(1.0, 1.0);
                [toView setFrame:containerView.bounds];


                toView.transform=CGAffineTransformMakeScale(0.9, 0.9);

                [ UIView animateWithDuration:director.animDuration
                    animations:^{

                    [fromView setFrame:CGRectMake(fromView.frame.size.width, fromView.frame.origin.y, fromView.frame.size.width, fromView.frame.size.height)];
                    toView.transform=CGAffineTransformMakeScale(1.0, 1.0);

                }completion:^(BOOL finished) {

                }];
            };
            [self.navigationController popViewControllerAnimated:YES];
            break;
        case  UIGestureRecognizerStateChanged:
        {
            //update percent for every step
            [animDirector setPercent:location.x/fullDistance];
            break;
        }
        case UIGestureRecognizerStateCancelled:
        case UIGestureRecognizerStateEnded: {

            BOOL didComplete=NO;
            if (pan.state==UIGestureRecognizerStateEnded){
            didComplete = (location.x/fullDistance)>0.5?YES:NO;
            }

            //and end interactive transition at state ended or canceled
            [animDirector endInteractiveTranscation:didComplete complition:^{

            }];
            animDirector = nil;
            self.navigationController.delegate = nil;

            break;
        }
        default: {

            break;
        }

    }
  }

}
``` 
So the main things that u should do:
* set director as interactive `animDirector.interactive=YES;`
* update director percent  `[animDirector setPercent:location.x/fullDistance];`
* run director interactive endingMethod: `[animDirector endInteractiveTranscation:didComplete complition:^{}];`

That looks nice but wee need some more, lets use the `@property (copy)UpdateBlock interactiveUpdateBlock`,
so we could create unique interactive transaction :

* Let add new method that will create maskLayer to "fromView"
``` objective-c
-(void)addMaskToView:(UIView*)view withPosition:(CGPoint)maskPosition{
    CAShapeLayer * maskLayer;
    if (!view.layer.mask) {
        maskLayer=[CAShapeLayer layer];
        view.layer.mask=maskLayer;
        maskLayer.fillRule=kCAFillRuleEvenOdd;
    }else{
        maskLayer=(CAShapeLayer*)view.layer.mask;
    }
        CGMutablePathRef path=CGPathCreateMutable();
        CGPathAddRect(path, nil, view.bounds);
        CGPathAddEllipseInRect(path, nil, CGRectMake(-28, maskPosition.y, 44, 44));
        maskLayer.path=path;
        CGPathRelease(path);
}
``` 
* make some updates in our UIGestureRecognizerStateEnded and UIGestureRecognizerStateChanged:
``` objective-c
//.....
case  UIGestureRecognizerStateChanged:
{

    animDirector.interactiveUpdateBlock=^(APTransitionDirector*director){
    UIView* fromView= [director fromView];
    [self addMaskToView:fromView withPosition:location];
    };
    //update percent for every step
    [animDirector setPercent:location.x/fullDistance];
    break;
}

case UIGestureRecognizerStateCancelled:
case UIGestureRecognizerStateEnded: {

    BOOL didComplete=NO;
    if (pan.state==UIGestureRecognizerStateEnded){
        didComplete = (location.x/fullDistance)>0.5?YES:NO;
    }   

        //and end interactive transition at state ended or canceled
    [animDirector endInteractiveTranscation:didComplete complition:^(APTransitionDirector*director){
    [director fromView].layer.mask=nil;

    }];
    animDirector = nil;
    self.navigationController.delegate = nil;
    break;
}
....
```
Ok, so now u have nice interactive and static transition.This code is in examples as well (BasicUsage-NavigationController)
u also could check some others examples there,like Basic Usage with TabbarController and ViewController.
<img src="https://github.com/apadalko/APTransitionDirector/blob/master/Examples/gifs/nav.gif" alt="Basic Usage NavigationController" width="480" height="360"/>

As u maybe noticed there are a little bit of extra code.To avoid this u could use APTransitionProtocol and `animDirector.delegate=someObject`;
Its pretty easy to use, u could see this in BasicUsage(APTransitionProtocol)-NavigationController.
As well u didnt need always to call `animDirector=nil` or `self.navigationController=nil` ,its up to you.(see the asicUsage(APTransitionProtocol)-NavigationController example)
 
## Category (APTransitions) & APTransitionProtocol


