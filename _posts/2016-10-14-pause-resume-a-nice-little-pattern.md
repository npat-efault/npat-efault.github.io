---
layout: post
title:  "Pause / Resume: A nile little pattern"
date:   2016-10-24 01:15:59
categories: golang
comments: yes
---

We want to "pause" and "resume" the operation of a cooperating
goroutine (controlled) from another goroutine (controlling). Once we
call "pause" the caller must block until the controlled goroutine is
in fact paused. Multiple controlling goroutines can call "pause"; the
same number must call "resume" before the contolling goroutine
resumes.

Here's an implementation using channels. It utilizes two channels of
the "empty" type. One carries pause requests, the other resume
requests.

    type ε struct{}

    type Pauser struct {
        npause  int
        cpause  chan ε
        cresume chan ε
    }
    
    func NewPauser() *Pauser {
        p := &Pauser{}
        p.cpause = make(chan ε)
        p.cresume = make(chan ε)
        return p
    }

These are the methods called by the controlling goroutines:

    func (p *Pauser) Pause() {
        p.cpause <- ε{}
    
    }

Method `Pause` asks the controlled goroutine to pause and blocks until
it does. *When Pause returns, we know the controlled goroutine is paused*.

    func (p *Pauser) Resume() {
        p.cresume <- ε{}
    }

Method `Resume` permits the controlled goroutine to resume. Returns
imediatelly. For the goroutine to remume, all Pause calls must be
matched by Resumes. It is illegal (panic) to call resume before
first calling Pause.

The following is the method called periodically by the controlled
goroutine. If a "pause" request is issued the method does not return
until the respective "resume" arrives.

    func (p *Pauser) CheckPause() {
        // We are running, p.nPause == 0, check if we should pause
        select {
        case <-p.cpause:
            p.npause++
        case <-p.cresume:
            panic("Pauser: Resume without pause")
        default:
            return
        }
    
    loop:
        // We are paused, p.nPause > 0, wait for pause / resume
        for {
            select {
            case <-p.cpause:
                p.npause++
            case <-p.cresume:
                p.npause--
                if p.npause == 0 {
                    break loop
                }
            }
        }
        // We are running again, p.nPause == 0
    }

That's all
