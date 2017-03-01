# ImageCache

let imageCache = NSCache<AnyObject, AnyObject>()

func associatedObject<ValueType: AnyObject>( base: AnyObject, key: UnsafePointer<UInt8>, initialiser: () -> ValueType) -> ValueType {
                if let associated = objc_getAssociatedObject(base, key) as? ValueType { return associated }
                let associated = initialiser()
                objc_setAssociatedObject(base, key, associated, .OBJC_ASSOCIATION_RETAIN)
                return associated
}
func associateObject<ValueType: AnyObject>( base: AnyObject, key: UnsafePointer<UInt8>, value: ValueType) {
        objc_setAssociatedObject(base, key, value, .OBJC_ASSOCIATION_RETAIN)
}

private var urlKey: UInt8 = 0

class URLString {
        var urlString = String()
}

extension UIImageView {
        
        var imageURL: URLString {
                get {
                        return associatedObject(base: self, key: &urlKey)
                        { return URLString() }
                }
                set { associateObject(base: self, key: &urlKey, value: newValue) }
        }
        
        func loadImageUsingUrlString(urlString: String) {
                imageURL.urlString = urlString
                guard let url = URL(string: urlString) else { return }
                image = nil
                if let imageFromCache = imageCache.object(forKey: urlString as AnyObject) as? UIImage {
                        self.image = imageFromCache
                        return
                }
                
                URLSession.shared.dataTask(with: url, completionHandler: { (data, respones, error) in
                        if error != nil {
                                return
                        }
                        if let imgData = data {
                                DispatchQueue.main.async(execute: {
                                        let imageToCache = UIImage(data: imgData)
                                        if self.imageURL.urlString == urlString {
                                                self.image = imageToCache
                                        }
                                        if let imageToBeCached = imageToCache {
                                                imageCache.setObject(imageToBeCached, forKey: urlString as AnyObject)
                                        }
                                })
                        }
                }).resume()
        }
}
