Coordinators, Routers, and Back Buttons
If you have stumbled upon this post you probably ran into the same issues I did when working with flow coordinators. If you are new to coordinators I suggest you read Soroush Khanlou’s Blog posts about coordinators. This work is the foundation of my understanding coordinators but upon trying to implement my own coordinators I ran into a huge flaw when dealing with the back button.
Horizontal vs. Vertical Flows
Child coordinators can represent either vertical or horizontal flows.
A child coordinator that you present is a vertical flow and upon its completion you dismiss it and deallocate the coordinator. An example of a vertical flow would be an authentication flow that is presented modally and only completes when the authentication is cancelled or the user is authenticated by a login or signup.
A child coordinator that you push to is a horizontal flow and upon its completion (either from an action or the back button) it is popped from the navigation controller stack and deallocated. For this to work, a parent and child coordinator need to share the same navigation controller. This case arose for me when pushing to a profile page. There were many other sub flows on this profile page so I could not simply instantiate a new profile view controller and push to it whenever I wished. I needed a reusable coordinator that I could push to.
Implementing this involves passing around a reference to a UINavigationController and a lot of messy code dealing with the navigation controller’s delegate to determine when the back button was pressed so you can deallocate the child coordinator. I began searching for a better way and I came across Andrey Panov’s coordinator implementation which included a new component to the architecture: the Router. You should read that as well because I will not cover all of it for the sake of brevity.
The router is a pretty simple concept. It is a class that wraps a UINavigationController and you pass it around between coordinators. It handles the physical navigation whereas the coordinator handles flow logic. There is one router for each horizontal flow and it may be shared by child coordinators for horizontal flows via dependency injection. Every time you have a new vertical flow (usually modally presenting a flow) this will require instantiating a new router.
The back button is the biggest pain when using either Khanlou or Andrey Panov’s approach for horizontal flows. Khanlou even had an entire follow up post regarding the use of Back Buttons and Coordinators together. I find these proposed solutions acceptable but still not ideal.
The first suggestion invloves creating a Navigation Controller class that is a subclass of UIViewController with the purpose of enclosing the navigationController, conforming to UINavigationControllerDelegate, and maintaining a mapping from view controller to coordinator so the coordinator may be deallocated upon popping the view controller. This seems like subclassing a UITabBarController to avoid putting the setup logic of our app in the AppDelegate. Does it work? Yes, but it doesn’t feel right. He goes on to say “My primary concern is that the NavigationController class, which is a view controller, knows about and has to deal with coordinators”. The second approach involves extending the coordinator to become the navigation controller delegate and then handling deallocation when this delegate method fires. The biggest drawback of this is that there is no way to handoff the responsibility of being the navigation controller delegate to subsequent child coordinators. If you push to a child flow and then push to another flow, the first flow receives the pop events for both of the flows. I’m not sure if this approach is feasible for pushing more than one level deep.
I think a combination of these two approaches would work better. What if we extend our Router class to conform to UINavigationControllerDelegate so the router can handle all the events for the navigation controller it wraps and delegate the responsibility fo what to do on the back button event back to the coordinator that pushed this coordinator in the first place?
In my opinion, the ideal solution should…
1. allow us to treat the back button of a horizontal flow like we would a cancel button on a vertical flow
2. avoid polluting navigation (router) with flow logic (coordinators)
3. delegate what to do when the back button is pressed (deallocation of child coordinator) back to the parent coordinator
4. provide a common interface for presenting and pushing coordinators and view controllers
5. allow us to push to subsequent horizontal flows with ease
In my opinion it should look something like this:
```swift
let coordinator = ProfileCoordinator(router: router, store: store, profile: profile)
addChild(coordinator)
router.push(coordinator) { [weak self, weak coordinator] in
    // This executes when the back button is pressed
    self?.removeChild(coordinator)
}
coordinator.start()
```
We should be able to interact with the router the same for both coordinators and view controllers:
```swift
let viewController = UIViewController()
router.push(viewController) {
    // This executes when the back button is pressed
}
```
So our push function needs to take a common type and our router needs to keep track of the callbacks so it can execute them when the back button is pressed.
Our common type is the Presentable protocol:
```swift
public protocol Presentable {
    func toPresentable() -> UIViewController
}
// UIViewController is already a presentable type
extension UIViewController: Presentable {
    public func toPresentable() -> UIViewController {
        return self
    }
}
```
Now our Router interface and implementation (This is a modified version of Andrey Panov’s implementation to provide back button support):
```swift
import UIKit

public protocol RouterType: class, Presentable {
    var navigationController: UINavigationController { get }
    var rootViewController: UIViewController { get }
    func present(_ module: Presentable, animated: Bool)
    func push(_ module: Presentable, animated: Bool, completion: (() -> Void)?)
    func popModule(animated: Bool)
    func dismissModule(animated: Bool, completion: (() -> Void)?)
    func setRootModule(_ module: Presentable, hideBar: Bool)
    func popToRootModule(animated: Bool)
}

final public class Router: NSObject, RouterType, UINavigationControllerDelegate {
    private var completions: [UIViewController : () -> Void]
    public var rootViewController: UIViewController...
    public unowned let navigationController: UINavigationController
    public init(navigationController: UINavigationController) {
        self.navigationController = navigationController
        self.completions = [:]
        super.init()
        self.navigationController.delegate = self
    }
    public func toPresentable() -> UIViewController {
        return navigationController
    }
    public func present(_ module: Presentable, animated: Bool) {
        guard let controller = module.toPresentable() else { 
            return
        }
        navigationController.present(controller, animated: animated, completion: nil)
    }
    public func dismissModule(animated: Bool, completion: (() -> Void)?) {
        navigationController.dismiss(animated: animated, completion: completion)
    }
    public func push(_ module: Presentable, animated: Bool = true, completion: (() -> Void)? = nil) {
        // Avoid pushing UINavigationController onto stack
        guard let controller = module.toPresentable(), 
            controller is UINavigationController == false else {
            return
        }
        if let completion = completion {
            completions[controller] = completion
        }
        navigationController.pushViewController(controller, animated: animated)
    }
    public func popModule(animated: Bool = true)  {
        
        if let controller = navigationController.popViewController(animated: animated) {
            runCompletion(for: controller)
        }
    }
    public func setRootModule(_ module: Presentable, hideBar: Bool){
        guard let controller = module.toPresentable() else { 
            return
        }
        navigationController.setViewControllers([controller], animated: false)
        navigationController.isNavigationBarHidden = hideBar
    }
    public func popToRootModule(animated: Bool) {
        if let controllers = navigationController.popToRootViewController(animated: animated) {
            controllers.forEach { runCompletion(for: $0) }
        }
    }
    fileprivate func runCompletion(for controller: UIViewController) {
        guard let completion = completions[controller] else {
            return
        }
        completion()
        completions.removeValue(forKey: controller)
    }
    // MARK: UINavigationControllerDelegate
    public func navigationController(_ navigationController: UINavigationController, didShow viewController: UIViewController, animated: Bool) {
        
        // Ensure the view controller is popping
        guard let poppedViewController = navigationController.transitionCoordinator?.viewController(forKey: .from), !navigationController.viewControllers.contains(poppedViewController) else {
             return
        }
        runCompletion(for: poppedViewController)
    }
}
```
Our router should be able to perform all possible navigation actions. It must also act as the delegate of the navigation controller so it can intercept back button presses and run the corresponding completion handler for the view controller that was popped. When a Presentable type is pushed, we gain access to the view controller to be pushed using the module.toPresentable() and store the completion handler in a dictionary with the key being the view controller. When a view controller is popped, either from the back button, the navigation controller delegate function determines which view controller was popped and executes the corresponding completion handler.
Now our Coordinator interface and implementation:
```swift
public protocol Coordinatable: class, Presentable {
    var router: RouterType { get }
    var onCompletion: (() -> Void)? { get set }
    func start()
}

open class Coordinator: NSObject, Coordinatable {
    
    public var childCoordinators: [Coordinator] = []
    public var router: Router
    public init(router: Router) {
        self.router = router
        super.init()
    }
    open var onCompletion: (() -> Void)?
    open func start() {}
    public func addChild(_ coordinator: Coordinator) {
        childCoordinators.append(coordinator)
    }
    public func removeChild(_ coordinator: Coordinator) {
        if let coordinator = coordinator, 
            let index = childCoordinators.index(of: coordinator) {
            childCoordinators.remove(at: index)
        }
    }
    // Make this function open so we can override it in a different module
    open func toPresentable() -> UIViewController {
        return router.toPresentable()
    }
}
```

Our Coordinator must implement Presentable so the router can present it. This default implementation will return the router’s presentable form which will just return the router’s underlying navigation controller. This works great when presenting a coordinator for a new vertical flow, but we cannot push such a coordinator. So to push child coordinators for horizontal flows we must override toPresentable() to give us a view controller instance that will work when we call router.push(coordinator). Note: In our Router, we guard to make sure the view controller we are pushing is not a UINavigationController.
So a subclass of Coordinator that we want to use for horizontal flows might look something like this:

```swift
import Foundation

open class ProfileCoordinator: Coordinator {
    fileprivate let store: StoreType
    fileprivate let profile: Profile
    lazy var viewController: ProfileViewController = {
        let viewModel = LocationProfileViewModel(profile: profile)
        return ProfileViewController(viewModel: viewModel)
    }()
    public init(router: RouterType, store: StoreType, profile: Profile) {
        self.store = store
        self.profile = profile
        super.init(router: router)
    }
    open override func start() {}
    open override func toPresentable() -> UIViewController {
        return viewController
    }
}
```
And voila! We can push this child coordinator from a parent coordinator and deallocate it with ease:
```swift
let coordinator = ProfileCoordinator(router: router, store: store, profile: profile)
addChild(coordinator)
router.push(coordinator) { [weak self, weak coordinator] in
    // This executes when the back button is pressed
    self?.removeChild(coordinator)
}
coordinator.start()
```
Hopefully this was helpful. This solution will work best if you use closures for the callbacks of your coordinators rather than delegates. Any feedback, questions, or criticism is welcome.
Check out CoordinatorKit for a demo.
