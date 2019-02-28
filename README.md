# AudioWorklet Unit Test Experiment with Jest

While building an app with AudioWorklets, [an example of which I made here](https://montag.gitlab.io/vue-audioworklet-demo/), I wondered how one might go about testing these things.

I looked at some of the testing tools/examples that are out there such as the [Web Platform Tests](https://github.com/web-platform-tests/wpt) and the [Web Audio Test Api](https://github.com/mohayonao/web-audio-test-api) as well as some [interesting Webkit tests](https://github.com/WebKit/webkit/blob/master/LayoutTests/webaudio/gain.html).

These tools had a few issues, notably they are mostly for testing the Web Audio Api itself, or an apps usage of it e.g. verify the audio graph(s). Secondarily, they all have their own api's and don't necessarily integrate with the standard tooling (mocha, jest, etc) that we might find in a web app. 

Yet, AudioWorklet processors have a simple and well-defined interface which makes their implementation a good candidate for unit testing entirely independent of the Web Audio Api itself. They are simply passed in Float32Arrays, doing some work, and return some Float32Arrays when they are done.  

### Prototype Goals ###

1. Mock the AudioWorkletProcessor superclass
2. Import and instantiate the worklet
3. Create realistic frames of audio data to feed into the worklet
4. Ability to modify parameters 
5. Verify that the output matches what it should with a decent amount of precision.
6. Do all of the above using the apps test suite (Jest) with the standard build chain (e.g. webpack4) 

### Steps ###

### 1. Mock the AudioWorkletProcessor superclass ###
1. `AudioWorkletProcessor` is the super class of all processors. This class is not defined in the main global scope (like `AudioWorkletNode` is). Nor is `registerProcessor`. So we need to mock both of those in order to instantiate our worklet processor. We also need to add anything the impl needs such as the message port (which we're not testing here)

/tests/unit/fixtures/AudioWorkletProcessor.js 
```ecmascript 6
    class MessagePort {
      constructor() {
      }
    
      postMessage(string) {
      }
    }
    
    export default class AudioWorkletProcessor {
      constructor() {
        this.port = new MessagePort()
      }
    }

    global.AudioWorkletProcessor = AudioWorkletProcessor
    global.registerProcessor = () => {}
```
This AudioWorkletProcessor.js mocks the necessary classes and functions needed in global scope where our tests will run, we just need to import it into our test.

/tests/unit/worklet.spec.js
```ecmascript 6
    import './fixtures/AudioWorkletProcessor.js'
```

### 2. Import and instantiate the worklet ###
2. Add 'export default' to the Worklet impl. Making it an es6 module doesn't effect the worklet when it's loaded into the AudioWorkletGlobalScope and it makes it easier to import into our tests.

/worklets/GainWorklet <--- the worklet under test
~~~ecmascript 6
    export default class GainWorklet extends AudioWorkletProcessor {
      ...
    }
~~~

/tests/unit/worklet.spec.js
```ecmascript 6
    import './fixtures/AudioWorkletProcessor.js'
    import GainWorklet from "@/worklet/GainWorklet";
```
 
### 3. Create realistic frames of audio to feed into the worklet ###

To create realistic audio data. I created a utility processor worklet that records 10 frames of input data, and fed that the sine wave output of an OscillatorNode at 440hz.

~~~ecmascript 6
  const oscillator = context.createOscillator()
  oscillator.frequency.value = 440
  oscillator.type = 'sine'
  oscillator.connect(captureProcessorNode)
  this.source = oscillator
  this.source.start()
~~~

Then in the capture processor, capture the first few input frames

~~~ecmascript 6
    if (this.count < 10) {
      this.port.postMessage({ msg: inputs })
    }
~~~

More complex tests could capture longer sets of data or could capture the output of the node-under-test as well for use in comparisons. 

source --> captureInputProcessor --> Processor Under Test --> captureOutputProcessor --> destination

For this experiment, the worklet is just two channel gain so capturing the output is not needed. See the `MockAudioData.js` file in `fixtures` for the data used in this test.

### 4. Ability to modify parameters ###

Since this processor is adjusting the gain from 0 to 1. I'd like to test the gain using a known set of inputs at 1.0, 0.5, and 0 and then make sure the output is as expected. That should verify that the processor is working on both channels relatively well. 

~~~ecmascript 6
test('test gain 1', () => {
    const worklet = new GainWorklet()
    const testGain = 1.0
    const inputs = MockAudio.inputs
    let outputs = MockAudio.outputs
    worklet.process(inputs, outputs, { gainChannel_0: [testGain], gainChannel_1: [testGain]})
    ...
  })
~~~

Above we can see that we are setting the `testGain` to 1.0, and passing in two mock `AudioParams` that the processor expects. It would also be possible to examine the parameters via `parameterDescriptors` for more complex or automated test generation. It should also be possible to mock the currentTime of the processor, and or to mock a-rate or k-rate params for more advance scenarios.

### 5. Verify that the output matches what it should with a decent amount of precision. ###

After the processor executes the data is available in `outputs`. 

~~~ecmascript 6
    worklet.process(inputs, outputs, { gainChannel_0: [testGain], gainChannel_1: [testGain]})
    const input = inputs[0]
    const output = outputs[0]
    for (let i = 0; i < input.length; ++i) {
      expect(MockAudio.framesMatch(output[i], input[i], testGain)).toBe(true)
    }
~~~

Here we loop through each channel and call a helper function to compare the two TypedArrays while also providing a multiplier function that duplicates what we expect the processor to be doing.  

Since this is a trivial example it's easy to duplicate the processor function. For more complex processors, it might make more sense to capture the output or even hash the outputs to save some space.

~~~ecmascript 6
/**
 * Compare two TypedArrays and apply multiplier
 */
export function framesMatch(resultFrame, expectedFrame, multiplier=1.0) {
  let rValues = resultFrame.values()
  let eValues = expectedFrame.values()
  let equal = true
  for (let r of rValues) {
    let e = eValues.next().value * multiplier
    if (r !== e) {
      equal = false
      break
    }
  }
  return equal
}
~~~

### 6. Do all of the above using the apps test suite (Jest) with the standard buildchain (e.g. webpack4) ###

As this is within a Jest unit test, we've integrated our test within the overall application's test suites. 

~~~
$ yarn test:unit
yarn run v1.13.0
warning ..\package.json: No license field
$ vue-cli-service test:unit
 PASS  tests/unit/worklet.spec.js
 PASS  tests/unit/mockAudiHelper.spec.js

Test Suites: 2 passed, 2 total
Tests:       7 passed, 7 total
Snapshots:   0 total
Time:        1.673s
Ran all test suites.
Done in 3.77s.
~~~

## Project setup
```
yarn install
```

### Run your unit tests
```
yarn run test:unit
```
