#!/usr/bin/python

import sys
import os
import logging
import subprocess
import shutil
import re
from optparse import OptionParser

from gi.repository import Gtk, Gdk, GLib, GdkPixbuf


DEFAULT_UI_FILE=''
DEFAULT_MUSIC_DIRECTORY="~/sheet-music"
METADATA_FILENAME="meta.txt"

logger = logging.getLogger('piano')

class Dialog:
    @staticmethod
    def confirm(question, parent=None):
        dlg = Gtk.MessageDialog(parent,
                                Gtk.DialogFlags.MODAL,
                                Gtk.MessageType.QUESTION,
                                Gtk.ButtonsType.YES_NO,
                                question)
        rv = dlg.run() == Gtk.ResponseType.YES
        dlg.destroy()
        return rv

class ScreenInfo:
    @staticmethod
    def get_dimensions():
        defaultScreen = Gdk.Screen.get_default()
        monitorGeometry = defaultScreen.get_monitor_geometry(
            defaultScreen.get_primary_monitor())
        return (float(monitorGeometry.x + monitorGeometry.width),
                float(monitorGeometry.y + monitorGeometry.height))

class MusicLibraryEntry:
    pass

class MusicLibrary:
    @staticmethod
    def is_valid_dir(path):
        return os.path.exists(os.path.join(path, METADATA_FILENAME))

    def __init__(self, directory):
        self.directory = directory
        logger.debug("Using '%s' as music directory", self.directory)
        self.entries = []
        self.refresh()

    def refresh(self):
        self.entries = []
        # TODO: realpath/abspath
        for d in [ os.path.join(self.directory, name) for name in 
                   os.listdir(self.directory) if os.path.isdir(os.path.join(
                    self.directory, name))]:
            if not MusicLibrary.is_valid_dir(d):
                logger.debug("Skipping %s", d)
                continue
            entry = MusicLibraryEntry()
            entry.title = None
            entry.dir = None
            entry.files = []
            try:
                with open(os.path.join(d, METADATA_FILENAME), 'r') as metadata:
                    lines = metadata.readlines()
                    entry.title = lines.pop(0).strip()
                    entry.dir = d
                    for i in lines:
                        filename = os.path.join(d, i.strip())
                        entry.files.append(filename)
            except IOError as e:
                logger.warn("Could not load metadata: " + d)

            if entry.title is not None:
                self.entries.append(entry)

    def __iter__(self):
        return iter(self.entries)

class Music:
    def __init__(self, library_entry):
        self.directory = library_entry.dir
        self.title = library_entry.title
        self.pixbufs = []
        (screen_w, screen_h) = ScreenInfo.get_dimensions()
        for filename in library_entry.files:
            pb = GdkPixbuf.Pixbuf.new_from_file(filename)
            if pb.get_height() > screen_h:
                factor = screen_h / pb.get_height()
                new_width = int(factor * pb.get_width())
                self.pixbufs.append(pb.scale_simple(new_width, 
                                                    screen_h,
                                                    GdkPixbuf.InterpType.BILINEAR))
            else:
                self.pixbufs.append(pb)
        self._page_count = len(self.pixbufs)
        if len(self.pixbufs) % 2 != 0:
            # Pad it with a page of grey
            self.pixbufs.append(self.pixbufs[-1].copy())
            self.pixbufs[-1].fill(0x696969ff)

        self.page = 0

    def get_displayed_pages_readable(self):
        l,r = (self.page + 1, self.page + 2)
        return "pages %d-%d of %d" % (l, l if r > self._page_count else r, self._page_count)

    def get_pages(self, **kwargs):
        if 'forw' in kwargs and kwargs['forw']:
            if self.page + 2 >= len(self.pixbufs):
                return None
            self.page += 2
        elif 'back' in kwargs and kwargs['back']:
            if self.page - 2 < 0:
                return None
            self.page -= 2
        # Range selector is length, not index
        return self.pixbufs[self.page:self.page + 2]

class MusicWindow:
    def __init__(self, options):
        self.music_dir = os.path.abspath(os.path.expanduser(options.music_dir))
        self.builder = Gtk.Builder()
        try:
            self.builder.add_from_file(options.ui_file)
            logger.debug("UI loaded")
        except GLib.GError as e:
            sys.exit("FATAL: Could not load UI: " + e.message)
        self.builder.connect_signals(self)
        self.library = MusicLibrary(self.music_dir)
        self.main_window = self.builder.get_object("main_window")
        self.music_window = self.builder.get_object("music_window")
        self.music_liststore = self.builder.get_object("music_liststore")
        rw = Gdk.get_default_root_window()
        rw.set_cursor(Gdk.Cursor(Gdk.CursorType.LEFT_PTR))
        self.main_window.show()
        self.refresh_library()

    def refresh_library(self, force=False):
        if force:
            self.library.refresh()
        self.music_liststore.clear()
        for entry in self.library:
            self.music_liststore.append((entry, entry.title))

    def get_selected_music_entry(self):
        treeview = self.builder.get_object('music_treeview')
        path = treeview.get_cursor()[0]
        if path is None:
            return None
        return self.music_liststore.get_value(
            self.music_liststore.get_iter(path), 0)

    def add_btn_clicked_cb(self, widget, data=None):
        dlg = self.builder.get_object("open_pdf_filechooser_dlg")
        rv = dlg.run()
        dlg.hide()
        if rv != Gtk.ResponseType.OK:
            return True
        entry = self.builder.get_object("title_entry")
        entry.set_text('')
        filename = dlg.get_filename()
        self.builder.get_object("filename_lbl").set_text(filename)
        dlg = self.builder.get_object("add_music_dlg")
        rv = dlg.run()
        dlg.hide()
        if rv != Gtk.ResponseType.OK:
            return True
        dpi = self.builder.get_object("dpi_spinbtn").get_value()
        #TODO: ensure directory doesn't exist, give options in dialog
        destdir = os.path.join(self.music_dir,
                               re.sub('\W+', '', entry.get_text()))
        try:
            os.mkdir(destdir)
        except OSError as e:
            logger.warn(e)
        
        try: 
            subprocess.check_call(['convert',
                                   '-quality',
                                   '00',
                                   '-density',
                                   "%{:.0f}x{:.0f}".format(dpi, dpi),
                                   filename,
                                   "{0}/page-%d.png".format(destdir)])
        except CalledProcessError as e:
            logger.warn("Oops: %s", str(e))
        files = os.listdir(destdir)
        try: 
            with open(os.path.join(destdir, METADATA_FILENAME), 'w') as f:
                f.write(entry.get_text() + "\n")
                f.writelines([ x + "\n" for x in sorted(files, key=lambda x: int(re.search('\d+', x).group(0)))])
        except IOError as e:
            logger.warn(e)
        self.refresh_library(force=True)
        

    def delete_btn_clicked_cb(self, widget, data=None):
        entry = self.get_selected_music_entry()
        if entry is None:
            widget.error_bell()
            return True
        if not Dialog.confirm(
            "Permanently delete sheet music for \"{0}\"?".format(
                entry.title),
            self.main_window):
            return True
        shutil.rmtree(entry.dir)
        self.refresh_library(force=True)

    def music_treeview_cursor_changed_cb(self, tree_view, data=None):
        tree_path = tree_view.get_cursor()[0]
        if tree_path is None:
            return False
        model = tree_view.get_model()
        tree_iter = model.get_iter(row)
        entry = model.get_value(tree_iter, 0)

    def music_treeview_row_activated_cb(self, tree_view, tree_path,
                                        col, data=None):
        model = tree_view.get_model()
        entry = model.get_value(model.get_iter(tree_path), 0)
        self.load_music(Music(entry))

    def main_window_destroy_cb(self, widget, data=None):
        Gtk.main_quit()

    def quit_btn_clicked_cb(self, widget):
        Gtk.main_quit()

    def load_music(self, music):
        self.music = music
        self.music_window.show_all()
        self.music_window.move(0,0)
        self.load_pages(self.music.get_pages())

    def load_pages(self, pixbufs):
        self.builder.get_object('music_title_lbl').set_text("%s (%s)" % (self.music.title, self.music.get_displayed_pages_readable()))
        if pixbufs is not None:
            self.builder.get_object("page1_img").set_from_pixbuf(pixbufs[0])
            self.builder.get_object("page2_img").set_from_pixbuf(pixbufs[1])
        
    def page1_eventbox_button_press_event_cb(self, widget, event):
        if event.button == 1:
            self.turn_pages(True)
            return True
        if event.button == 3:
            self.music_window.hide()
        return False

    def page2_eventbox_button_press_event_cb(self, widget, event):
        if event.button == 1:
            self.turn_pages()
            return True
        if event.button == 3:
            self.music_window.hide()
        return False

    def turn_pages(self, backwards=False):
        self.load_pages(self.music.get_pages(forw=not backwards,
                                             back=backwards))
        
    def music_window_key_press_event_cb(self, widget, event):
        if event.keyval in (Gdk.KEY_Q, Gdk.KEY_q):
            self.music_window.hide()
        if event.keyval in (Gdk.KEY_plus, Gdk.KEY_KP_Add):
            self.turn_pages()
        elif event.keyval in (Gdk.KEY_minus, Gdk.KEY_KP_Subtract):
            self.turn_pages(True)


if __name__ == '__main__':
    parser = OptionParser()
    parser.set_defaults(debug=False,
                        ui_file=DEFAULT_UI_FILE,
                        music_dir=DEFAULT_MUSIC_DIRECTORY)
    parser.add_option("--debug", action="store_true", dest="debug")
    parser.add_option("--ui", action="store", type="string",
                      dest="ui_file")
    parser.add_option("--dir", action="store", dest="music_dir");
    (options, args) = parser.parse_args()
    if options.debug:
        logging.basicConfig(level=logging.DEBUG)
    win = MusicWindow(options)
    Gtk.main()

