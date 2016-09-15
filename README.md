# !!! WIP !!!

# bosh-inception-day2
This is the second of a multi-day bosh inception. The second day plays off of the first, so completing the bosh-inception-day1 is a prerequisite. The goal of this second day is to deploy a director, create a bosh release, and create a manifest for our new release, and finally deploy our release.

## Agenda (Rough)

| Time | Topic                                                                                         |
| ---- | --------------------------------------------------------------------------------------------- |
| 1000 | Go over environment                                                                           |
| 1010 | Ensure director is setup and running                                                          |
| 1020 | What is a bosh release                                                                        |
| 1030 | What are we going to deploy (gogs.io)                                                         |
| 1040 | Determine dependencies                                                                        |
| 1045 | Create release skeleton (lab)                                                                 |
| 1100 | Add dependencies to release (lab)                                                             |
| 1105 | Create job (lab)                                                                              |
| 1115 | Manifest Generation                                                                           |
| 1145 | Deploy Release                                                                  |
| 1200 |                                                                            |

## Prep

In order to perform the environment creation steps you will need to have access to AWS along with AWS Access and Secret Keys. You will also have to create a key-pair on AWS and have downloaded the resultant pem file. You should have also completed bosh-inception-day1.

## Slide Deck

To run the slide deck go into the `presentation` directory and execute `python -m SimpleHTTPServer`
