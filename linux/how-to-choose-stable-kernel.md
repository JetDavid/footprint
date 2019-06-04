# How to choose stable kernel

## Key Points

- more stable
- reduce maintenance costs
- advanced features (low priority)

## Summary in front


## How does Linux kernel devel and release model work?

### Commit kernel changes into kernel source tree

- 2017, developers: over 4,300 developers from 530 companies
- 2017, what were the developers done: 5 different kernel releases.
- 2017, each release containing between 12,000 and 14,500 different changes.
- On average, 8.5 changes are accepted into linux kernel every hour, every hour of the day.
- Non-scientific study shows that each changes needs to be submitted 2-3 times before it is accepted into the kernel source tree
- Every submitting is rigorous review and testing process that all kernel chagnes are put thought.

### Release kernel

- After Dec, 2003, 2.6 kernel. community switch dev and stable branch to "stable only" branch
- Solve very long release cycle (3 years) and the stuggle to maintain two different branches at the same time.
- July 2011, linus changed linux kernel version to 3.x after 2.6.39 kernel was released.
- April 2015, change the version from 3.19 relase to 4.0 release number.
- Every week, a new kernel will be released.

#### Stable release

- Stable release model started in 2005
- Stable relesae was based on Linus release. e.g. 4.9
- Stable release version is 4.9.x
- One single kernel developer is responsible for picking the needed patches for one stable release, and doing the review/release process.
- Stable kernels are maintained for as long as the current development cycle is happening. After Linus releases a new kernel, the previous stable kernel release tree is stopped and users must move to the newer released kernel.
- 10-15 patches will be accepted per day for stable relesae.

#### Long-Term stable kernel

- Because of the few months of stable release, LTS came about.
- First LTS is 2.6.16, in 2006
- Once a year a new LTS kernel will be picked.
- Every LTS kernel will be maintained by the kernel community for at lease 2 years.
- After 4.1.y LTS, no new LTS Kernel being based on an individual distribution's needs was created. Because many LTS kernel were released in the same year causing lots of confusing for vendors and users. But this have already delay many development plans of SoC companies.(Because the new chipsets need to depend on new kernel ---- The Andy Beer's law)
- The older LTS release (older than the releases of current year) are still being maintained by some kernel developers with a slower release.
- 9-10 patches will be accepted per day for LTS relesae. The number of patches is fluctuating.
- Older LTS release will apply less patches.

#### Stable kernel patch rules

[The rules of how to patch stable kernel](https://www.kernel.org/doc/html/latest/process/stable-kernel-rules.html)

## Kernel Updates

- Linux Kernel community has promised that no upgrade will break anything that is currently working in a previous release.
- For SoC, every upgrade patches are very huge cause of their large size and heavy modification of architecture specific and so on.
- Merging SoC code into kernel release is a difficut work because that the project-planing and business priorities of these compaines prevent them from dedicating resources to work on it.
- SoC's LTS kernel support is still a process that cannot be standardized for a long time.
- The Android project has standardized on the LTS kernels as a “minimum kernel version requirement”.

## Reference

- [Linux Kernel Release Model](http://kroah.com/log/blog/2018/02/05/linux-kernel-release-model/)
- [Active kernel releases](https://www.kernel.org/category/releases.html)