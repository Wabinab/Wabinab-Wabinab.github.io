# Setting Up Bend On WSL2: Fix This For Me!
computer_bugs technology cuda

A few days ago, one heard of this computer program called [Bend](https://github.com/HigherOrderCO/Bend/tree/main) that offers "whatever parallel will run in parallel" so you don't have to write threading and all that, so one decide to try it out today. Unfortunately, just installing it is a pain in the arse. More importantly, installing it for the current version doesn't run as expected. The versions as of writing are:

```bash
hvm = 2.0.19
bend-lang = 0.2.33
```

So, here is a walkthrough. First, this installation is done in WSL2, so that's different from actually installing it in Linux. One actually had a WSL2 with Ubuntu 20.04 installed, and CUDA v11. That, unfortunately, won't run; as mentioned in the README.md of the page, as of writing, it requires CUDA v12. So, setting out to install CUDA is a pain in itself, but it was nevertheless successful after certain trials. 

But there's a problem. Running `nvcc --version` still gives CUDA v11. There must be an error with the PATH and LD_LIBRARY_PATH. So, one had to deal with that by adding such lines to `sudo nano ~/.bashrc`

```bash
export PATH=/usr/local/cuda-12.5/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.5/lib64
```

Now that nvcc shows the correct version, we go ahead and install hvm and bend. Running them was okay, except, when it comes to running with cuda, it jammed there. The exact problem wasn't known, but it's as if you execute a while loop that do nothing infinitely. 

So, one set out to search for a solution. The [first thread on the issues page of Github](https://github.com/HigherOrderCO/Bend/issues/538) that one come across, after reading through the not-that-long thread, one found [one relevant solution](https://github.com/HigherOrderCO/Bend/issues/538#issuecomment-2156120451) that might work. Updating Ubuntu from 20.04 to 22.04 huh? 

Unfortunately, downloading WSL from the store and update, installing it and run `lsb_release -a`, it still shows Ubuntu 20.04. What happened? At first, one thought one might had installed the wrong version, installing `Ubuntu` from Microsoft Store, so one changed to install `Ubuntu 20.04.3 LTS` instead. Upon installing, one realized one might not have enough space on C drive, so one deleted it and install it on D instead using some other techniques like downloading from the `aka` website Microsoft owns. After some pain in the arse Powershell command one saw online, it finally install complete, and startup, check again, `lsb_release -a`. It shows... Ubuntu 20.04!!! What happened? 

After painful hours, one realized it's because they kept referring to the same vhdx file. Unfortunately, the original ubuntu v20.04 wasn't installed from Microsoft Store, so one had to make a zip copy of it first before deleting the whole folder. That free up some space for me to install on C drive. However, one already had it on D drive, so when one click, it install a new vhdx file on D drive. Ok, whatever, it saves my space on C drive anyways. (Also, one uses Ubuntu 22.04.1 LTS instead of 22.04.3 because that's what Microsoft offers, so, not really have much simpler choices, do one?)

Check `lsb_release -a` gives the correct version. Simultaneously, while one deletes 20.04, one was downloading 24.04, so that got installed in C drive. Now, one had both wsl running, 22 and 24. One can now test on both. With a new drive, one had to go through the pain of installing nvcc again (the problem is it takes very long to go from running the command to the accept EULA agreement page, and from agreement page to finish install page). After installing in both VM, and other required stuffs (including forget to install gcc and g++ and go back to install before running the installation of nvcc process again, etc.) Now, for bend. Let's run it again; and... Violà, the problem persists. 

That really doesn't explain. Why would it stuck there forever? Scrolling through the thread, one saw a link to [another thread](https://github.com/HigherOrderCO/Bend/issues/364), and there was [a solution](https://github.com/HigherOrderCO/Bend/issues/364#issuecomment-2123806021). Now, this is related to one, because one's using GTX 1080 Ti instead of RTX 3070; and that had to do with reducing the memory set or something. Ok, one don't understand what the code means, but one don't need to understand it anyways; just follow the trick to fix the code and install `hvm` from there. After some pain in the arse doing it to both 22 and 24 Ubuntu, one tried bend again, and... nope, it's not fix. Still jam there. 

Just to repeat what he did in case the thread is gone for whatever reason (and changing the hvm to v2.0.19): 
```bash
mkdir ~/hvmtmp
cd ~/hvmtmp
cargo init
cargo add hvm@=2.0.19
cargo vendor vendor
cd vendor/hvm
code .  # open in VS Code, my editor of choice.
```

Edit `src/hvm.cu` `L_NODE_LEN` and`L_VARS_LEN`: 
```cu
// Local Net
const u32 L_NODE_LEN = 0x2000/4;
const u32 L_VARS_LEN = 0x2000/4;
struct LNet {
  Pair node_buf[L_NODE_LEN];
  Port vars_buf[L_VARS_LEN];
};
```

Then install: 
```bash
cargo install --path .
```

Now, clearly, almost all specs had been similar to the suggested solution. What hadn't been fix yet? If you read my article on [DPC Watchdog Violation](https://wabinab.github.io/article?filename=2024-06-09-DPC_Watchdog_Violation.md) before, you'd know one really, really, really, don't want to change the Nvidia Driver version in case it fails again. But this make one no choice. That's the only difference left: they're using Driver Version 555, but one's using 536. Ok, so we go, update driver. Whether or not it'll blue screen in the future, one don't know, as one just installed it today, and that requires a few day to a few weeks to examine the hypothesis. Anyways, after installation, we'll verify it with `nvidia-smi`. Running that on powershell, it's okay. Running that on wsl2, both Ubuntu 22 and 24 gives `segmentation fault` problem. 

Let's go back in time a bit. Before that happens, one actually try install `hvm v2.0.13` as mentioned by the author, but it has an error: 

```bash
CUDA error: the provided PTX was compiled with an unsupported toolchain
```

Searching through that error, it suggested updating driver, updating CUDA, updating whatever and then **restart** the computer. Ok, so the keyword restart can't be removed from one's mind. One go ahead and restart the computer. Upon finish starting, running `nvidia-smi` on both Ubuntu returns as expected. 

Let's go back to run bend now. And... after a few seconds, the result comes back! Violà! That's the first step to fix, but it took too long. Then, one goes to make some comments on the thread to show one fixed the problem, then come back and try again. This time, it took far less time to do the arithmetic. Ok, so that's more expected. 

So far, we only try a simple.bend program suggested in the thread: 
```python
def main():
  return (1 + 1)
```

Next, we'll try something harder. Let's deal with the `parallel_sum.bend` suggested as hello world: 
```python
# Defines the function Sum with two parameters: start and target
def Sum(start, target):
  if start == target:
    # If the value of start is the same as target, returns start.
    return start
  else:
    # If start is not equal to target, calculate the midpoint (half),
    # then recursively call Sum on both halves.
    half = (start + target) / 2
    left = Sum(start, half)  # (Start -> Half)
    right = Sum(half + 1, target)
    return left + right

# A parallelizable sum of numbers from 1 to 1000000
def main():
  # This translates to (((1 + 2) + (3 + 4)) + ... (999999 + 1000000)...)
  return Sum(1, 1_000_000)
```

Trying to run: 
```bash
$ bend run-c parallel_sum.bend -s
Result: 5908768
- ITRS: 45999971
- TIME: 0.69s
- MIPS: 66.89

$ bend run-cu parallel_sum.bend -s
Result: 5908768
- ITRS: 45983587
- LEAK: 37606783
- TIME: 0.83s
- MIPS: 55.62
```

Notice the LEAK is high, and notice the TIME to run on CPU parallel is less than on GPU. Although the problem is now solved, the results aren't enticing. Anyway, perhaps one can't suppose that; it may be designed to run on an RTX 4090 instead of 1080 Ti anyways. 

That's it for today. 

### References: 
1. https://github.com/HigherOrderCO/Bend/issues/538#issuecomment-2156120451
2. https://github.com/HigherOrderCO/Bend/issues/364#issuecomment-2123806021

### Specs:
- Windows 10 Pro
- i5-7400
- GTX 1080 Ti

[One now offers casual writing service to the public. Check it out!](https://www.fiverr.com/s/D84XrA)