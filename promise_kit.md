# PromiseKit
## Highlights
### Simple promise
```swift
func getWeather() -> Promise<Weather> {
  return Promise { fulfill, reject in
	  if ... {
      fulfill(weather)
    } else {
      reject(error)
    }
  }
}
```

### Promise SDK methods
#### URLDataPromise
```swift
let session = URLSession.shared
let dataPromise: URLDataPromise = session.dataTask(with: request)
dataPromise.asDictionary().then { }
```
#### LocationPromise
```swift
CLLocationManager.promise().then { }
```

### Threading
```swift
// Background thread
let session = URLSession.shared
let dataPromise: URLDataPromise = session.dataTask(with: request)
let backgroundQ = DispatchQueue.global(qos: .background)
_ = dataPromise.then(on: backgroundQ) { data -> Void in
  let image = UIImage(data: data)!
  fulfill(image)
}.catch(execute: fail)

// Main thread
{ }.then(on: DispatchQueue.main) { icon -> Void in
  self.iconImageView.image = icon
}
```
### Wrap
```swift
func getIcon(named iconName: String) -> Promise<UIImage> {
	// 1
  return wrap {
    // 2
    getFile(named: iconName, completion: $0)
    
    } .then { image in
      if image == nil {
        // 3
        return self.getIconFromNetwork(named: iconName)
        
      } else {
        // 4
        return Promise(value: image!)
      }
  }
}

func getIconFromNetwork(named iconName: String) -> Promise<UIImage> {
  let urlString = "http://openweathermap.org/img/w/\(iconName).png"
  let url = URL(string: urlString)!
  let request = URLRequest(url: url)
  
  let session = URLSession.shared
  let dataPromise: URLDataPromise = session.dataTask(with: request)
  return dataPromise.then(on: DispatchQueue.global(qos: .background)) { data -> Promise<UIImage> in
    return firstly { Void in
      return wrap { self.saveFile(named: iconName, data: data, completion: $0)}
      }.then { Void -> Promise<UIImage> in
        let image = UIImage(data: data)!
        return Promise(value: image)
    }
  }
}
```

1. `wrap(body:)` takes any function that follows one of several completion handler idioms and wraps it in a promise.
2. `getFile(named: completion:)` has a completion parameter of `@escaping (UIImage?) -> Void`, which becomes a Promise. It’s in wrap‘s body block where the original function is called, passing in the completion parameter.
3. Here, if the icon was not located locally, a promise to fetch it over the network is returned.
4. But if the image is available, it is returned in a value promise.

### Parallel
```swift
// race(promises:)
// The then block will only be called when the first of those promises are fulfilled. In theory, this should be a random choice due to variation in server conditions

  let weatherPromises = randomCities.map { weatherAPI.getWeather(latitude: $0.2, longitude: $0.3) }
  _ = race(promises: weatherPromises).then { weather -> Void in
    self.placeLabel.text = weather.name
    self.updateUIWithWeather(weather: weather)
    self.iconImageView.image = nil
  }
```

The first, `race` returns a promise that is fulfilled when the first of a group of promises is fulfilled. In essence, the first one completed is the winner.

The other two functions are `when` and `join`. Those fulfill after all the specified promises are fulfilled. Where these two differ is in the rejected case. `join` always waits for all the promises to complete before rejected if one of them rejects, but `when(fulfilled:)` rejects as soon as any one of the promises rejects. There’s also a `when(resolved:)` that waits for all the promises to complete, but always calls the then block and never the catch.

## Links
[Getting Started with PromiseKit](https://www.raywenderlich.com/145683/getting-started-promises-promisekit)
https://videos.raywenderlich.com/screencasts/705-getting-started-with-promisekit
https://videos.raywenderlich.com/screencasts/735-promisekit-background-promises

#ios/promise-kit