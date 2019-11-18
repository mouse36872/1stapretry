# 1stapptry
Trying
heroku create
Creating app... done, ⬢ thawing-inlet-61413
https://thawing-inlet-61413.herokuapp.com/ | https://git.heroku.com/thawing-inlet-61413.git

let n = vDSP_Length(2048)

let frequencies: [Float] = [1, 5, 25, 30, 75, 100,
                            300, 500, 512, 1023]

let tau: Float = .pi * 2
let signal: [Float] = (0 ... n).map { index in
    frequencies.reduce(0) { accumulator, frequency in
        let normalizedIndex = Float(index) / Float(n)
        return accumulator + sin(normalizedIndex * frequency * tau)
    }
}

let observed: [DSPComplex] = stride(from: 0, to: Int(n), by: 2).map {
    return DSPComplex(real: signal[$0],
                      imag: signal[$0.advanced(by: 1)])
}
let halfN = Int(n / 2)

var forwardInputReal = [Float](repeating: 0, count: halfN)
var forwardInputImag = [Float](repeating: 0, count: halfN)

var forwardInput = DSPSplitComplex(realp: &forwardInputReal,
                                   imagp: &forwardInputImag)

vDSP_ctoz(observed, 2,
          &forwardInput, 1,
          vDSP_Length(halfN))

let log2n = vDSP_Length(log2(Float(n)))

guard let fftSetUp = vDSP_create_fftsetup(
    log2n,
    FFTRadix(kFFTRadix2)) else {
        fatalError("Can't create FFT setup.")
}
defer {
    vDSP_destroy_fftsetup(fftSetUp)
}
var forwardOutputReal = [Float](repeating: 0, count: halfN)
var forwardOutputImag = [Float](repeating: 0, count: halfN)
var forwardOutput = DSPSplitComplex(realp: &forwardOutputReal,
                                    imagp: &forwardOutputImag)

vDSP_fft_zrop(fftSetUp,
              &forwardInput, 1,
              &forwardOutput, 1,
              log2n,
              FFTDirection(kFFTDirection_Forward))
let componentFrequencies = forwardOutputImag.enumerated().filter {
    $0.element < -1
}.map {
    return $0.offset
}
        
// Prints "[1, 5, 25, 30, 75, 100, 300, 500, 512, 1023]"
print(componentFrequencies)
var inverseOutputReal = [Float](repeating: 0, count: halfN)
var inverseOutputImag = [Float](repeating: 0, count: halfN)

var inverseOutput = DSPSplitComplex(realp: &inverseOutputReal,
                                    imagp: &inverseOutputImag)

vDSP_fft_zrop(fftSetUp,
              &forwardOutput, 1,
              &inverseOutput, 1,
              log2n,
              FFTDirection(kFFTDirection_Inverse))
var recreatedObserved = [DSPComplex](repeating: DSPComplex(real: 0, imag: 0),
                                     count: halfN)

vDSP_ztoc(&inverseOutput, 1,
          &recreatedObserved, 2,
          vDSP_Length(halfN))

let scale = 1 / Float(n * 2)
let recreatedSignal = recreatedObserved.map {
    [$0.real * scale, $0.imag * scale]
}.flatMap {
    return $0
}

