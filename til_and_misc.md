# TIL (& day to day problems the Internet never heard of)

### 1. Always use *MOUSEEVENTF_VIRTUALDESK* when using *MOUSEEVENTF_MOVE* to simulate moving the mouse cursor on Windows

I'm writing a Teamviewer clone in WPF. I managed to share a video feed and basic clicks and keyboard inputs from the controlling side to the sharing side. The only problem left was to move the mouse. Easy peasy, right? (foreshadowing)\
As always, it immediately came to mind that I want to be able to select the screen I'm sharing. Considering that I'm running a dual-monitor setup, I can test without going blindly into the matter at hand, or worse, or worse to trust ChatGPT. So, reading the [documentation](https://learn.microsoft.com/en-gb/windows/win32/api/winuser/ns-winuser-mouseinput), M$ (Microsoft) writes that:
>If the mouse has moved, indicated by MOUSEEVENTF_MOVE, dx and dy specify information about that movement. The information is specified as absolute or relative integer values.\
>If MOUSEEVENTF_ABSOLUTE value is specified, dx and dy contain normalized absolute coordinates between 0 and 65,535. [..] In a multimonitor system, the coordinates map to the primary monitor.

It seems like exactly what I'd want. But just in case, let's read the other flag that'll fit with moving the cursor:
> If MOUSEEVENTF_VIRTUALDESK is specified, the coordinates map to the entire virtual desktop.

It doesn't tell much why I would map to the entire virtual screen, so I'd sleep over it. In any case, it doesn't sound important... (foreshadowing)

You'd think you can get away by simply using MOUSEEVENTF_ABSOLUTE and moving the cursor according to the top and left bounds assigned to the shared monitor. You'd be right only for the 98% of the users that don't set custom resolutions and that don't set scaling factors for their displays. But I'm inside that made-up 2%, and I can't be oblivious to specific, unlikely use cases for which my proof-of-concept app wouldn't even see the bytes of another PC besides mine. I know it's a small project, and I can undoubtedly and successfully say "it works on my machine", but I wanted this project to go along with as many best practices as I could. Anyway, back to the subject at hand.

Some details<sup>1</sup>:
```
\\.\DISPLAY1 (primary)
	Real resolution: 1920x1080
	Scaled resolution: 1722x968
	Bounds: {X=0,Y=0,Width=1920,Height=1080}
\\.\DISPLAY3
	Real resolution: 2176x1224 (upscaled from the original resolution of 1920x1080)
	Scaled resolution: 1952x1098
	Bounds: {X=-2176,Y=-138,Width=2176,Height=1224}
```
![image](https://github.com/user-attachments/assets/9a6a634d-9b5b-4127-bcb7-86c3921c2a7d)


I started integrating the code and began testing:
```c#
private void MouseMoveReceived(double normalizedX, double normalizedY) {
	int targetX = SharedScreen.Bounds.Left + (int)(normalizedX * SharedScreen.Bounds.Width);
	int targetY = SharedScreen.Bounds.Top + (int)(normalizedY * SharedScreen.Bounds.Height);

	var inputs = new NativeMethods.INPUT[1];
	inputs[0].type = INPUT_MOUSE;
	inputs[0].u.mi.dx = targetX * 65535 / SharedScreen.Bounds.Width;
	inputs[0].u.mi.dy = targetY * 65535 / SharedScreen.Bounds.Width;
	inputs[0].u.mi.dwFlags = NativeMethods.MOUSEEVENTF_ABSOLUTE | NativeMethods.MOUSEEVENTF_MOVE;
	_ = NativeMethods.SendInput((uint)inputs.Length, inputs, Marshal.SizeOf(typeof(NativeMethods.INPUT)));
}
```
This code worked perfectly on my primary screen, but when I started testing on my secondary screen... it's here where I lost several days and my patience.\
_The problem_: setting `inputs[0].u.mi.dx` to 0 would land the cursor to the left border of my left screen, and setting it to -65535 would land it to -1920, offset with 256 pixels to the right where it's supposed to land.\
I thought it might be a problem with either my second monitor resolution or scaling, and surely enough, the resolution was the culprit<sup>2</sup>, now -65535 lands on the left margin of the screen, but this doesn't suffice, I can't stand a 24" monitor with the specific zoom of a 1080p display. So I rolled back the resolution and started searching again.

After some time, it struck me: what if I use **MOUSEEVENTF_VIRTUALDESK**? After all, moving through the virtual desktop area is trivial. I added the flag and... bingo!


Let's come back to something I mentioned above: *If MOUSEEVENTF_ABSOLUTE value is specified, dx and dy contain normalized absolute coordinates between 0 and 65,535. [..] In a multimonitor system, the coordinates map to the primary monitor.* But what they don't mention is that going beyond 65,535 in X axis, for example, is perfectly legal. This time I moved my second monitor to the right and started fiddling with that 0-65,535 range. So I doubled it, thinking that it would treat my primary screen as left margin=0, right margin=65,535 and the second monitor having the left margin=65,536 and the right margin=65,535\*2. But it seems like 65,535\*2 maps again (like when I had the monitor on the left and went to -65,535) to some offset to the actual (second) screen margin. So I started adding increments of 1000/100/10 until I arrived at 65535\*2 + 8720. It's what I feared the most, 0-65,535 maps the width/height of my primary monitor, and the adjacent monitors' range is a non whole factor, dependent on the primary display's resolution.

Moral of the story: don't add increments of 1000/100/10 to find out useless things when you already know you should've used MOUSEEVENTF_VIRTUALDESK in the first place.


<sub>1. The way Windows maps the virtual desktop to actual display pixels makes those details irrelevant, apart from the following note.</sub>\
<sub>2. As we found out, my main screen's resolution is 1920x1080 and so is my second monitor's, that's why -65535 mapped perfectly to the left edge of the screen while switching to the default resolution. I wonder if Microsoft noted this in their _Windows Internals_ books.</sub> 
